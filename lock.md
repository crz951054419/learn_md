title: 锁
date: 2018-08-31
author: crz
tags:
categories: lock
permalink: Lock/lock

---



- [1. 对象头](#1)
- [2. synchronized](#2)
  + [2.1 偏向锁](#2.1)
  + [2.2 轻量级锁](#2.2)
  + [2.3 自旋锁](#2.3)
  + [2.4 重量级锁](#2.4)
  + [2.5 锁膨胀](#2.5)
  + [2.6 锁消除](#2.6)
- [3. lock](#3)
- [4. 锁优化](#4)
- [5. 乐观锁/悲观锁](#5)
- [6. 分布式锁](#6)


---

# <span id="1">1. 对象头</span>

HotSpot虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。
HotSpot虚拟机的对象头(Object Header)包括两部分信息，第一部分用于存储对象自身的运行时数据， 如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等等，这部分数据的长度在32位和64位的虚拟机（暂不考虑开启压缩指针的场景）中分别为32个和64个Bits，官方称它为“Mark Word”。

![Image text](http://7xq3rs.com2.z0.glb.qiniucdn.com/15356087533772ieTsLlG_对象头.jpeg)

src/share/vm/oops/markOop.hpp
 
```java
//32 bits:
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//             size:32 ------------------------------------------>| (CMS free block)
//             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)

//64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
//
//  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
//  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
//  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
//  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)


//  - the two lock bits are used to describe three states: locked/unlocked and monitor.
//
//    [ptr             | 00]  locked             ptr points to real header on stack
//    [header      | 0 | 01]  unlocked           regular object header
//    [ptr             | 10]  monitor            inflated lock (header is wapped out)
//    [ptr             | 11]  marked             used by markSweep to mark an object
//                                               not valid at any other time


```




# <span id="2">2. synchronized</span>

Java 虚拟机中的同步(Synchronization)基于进入和监视器锁(Monitor)对象实现， 无论是显式同步(有明确的 monitorenter 和 monitorexit 指令,即同步代码块)还是隐式同步都是如此。在 Java 语言中，同步用的最多的地方可能是被 synchronized 修饰的同步方法。同步方法 并不是由 monitorenter 和 monitorexit 指令来实现同步的，而是由方法调用指令读取运行时常量池中方法的 ACC_SYNCHRONIZED 标志来隐式实现的 

非静态
 ![Image text]( http://7xq3rs.com2.z0.glb.qiniucdn.com/1535621010389DgZmnIeL_普通sy.png)
 静态
![Image text]( http://7xq3rs.com2.z0.glb.qiniucdn.com/1535621010342K6BiO07B_静态sy.png)

  在synchronized中主要涉及的锁类别有

* 偏向锁
* 轻量级锁
* 自旋锁
* 重量级锁


## <span id="2.1">2.1 偏向锁</span>

偏向锁(bias lock) hotspot默认支持 MarkWord的一个比特位标识该对象偏向锁是否被使用或者是否被禁止。

* 0-》则该对象未被锁定
* 1-》
     * 匿名偏向 thread_ptr为NULL(0),意味着没有线程偏向这个锁对象，
      当第一个线程去尝试获取锁，会遇到这个情况，使用原子cas可将该对象绑定于当前线程
     * 可重偏向 偏向锁epoch字端无效(  与锁对象对应的klass的mark_prototype的epoch值不匹配).
      下一个试图获取锁的对象会面临这个情况，使用原子CAS指令可将该锁对象绑定于当前线程。在批量重偏向的操作中，未被持有的锁对象都被至于这个状态，以便允许被快速重偏向。
      * 3.已偏向 这种状态下，thread ptr非空，且epoch为有效值——意味着其他线程正在只有这个锁对象。
    
jvm启动4秒这个feature是禁止的，jdk1.6后偏向是默认开启的 -XX:+UseBiasedLocking      

获取偏向锁的步骤：
* 1.验证对象的bias位
      如果是0，则该对象不可偏向，应该使用轻量级锁算法。
* 2.验证对象所属InstanceKlass的prototype的bias位
      确认prototype的bias为是否被设置。如果没有设置，则该类所有对象全部不允许被偏向锁定；并且该类所有对象的bias位都需要被重置，使用轻量级锁替换。
* 3.校验epoch位
      校验对象的MarkWord的epoch位是否与该对象所属InstanceKlass的prototype的MarkWord的epoch匹配。如果不匹配，则表明偏向已过期，需要重新偏向。这种情况，偏向线程可以简单地使用原子CAS指令重新偏向于这个锁对象。
* 4.校验owner线程
      比较偏向线程ID与当前线程ID。如果匹配，则表明当前线程已经获得了偏向，可以安全返回。如果不匹配，对象锁被假定为匿名偏向状态，当前线程应该尝试使用CAS指令获得偏向。如果失败的话，就尝试撤销(很可能引入安全点)，然后回退到轻量级锁；如果成功，当前线程成功获得偏向，可直接返回。

偏向锁的释放

* 偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动去释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态，撤销偏向锁后恢复到未锁定（标志位为“01”）或轻量级锁（标志位为“00”）的状态。

##<span id="2.2">2.2 轻量级锁 </span>
  
  获取轻量级锁的步骤

  *  1.在代码入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈 帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，官方称之为 Displaced Mark Word。
  * 2.拷贝对象头中的Mark Word复制到锁记录中。
  * 3.拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock record里的owner指针指向object mark word。如果更新成功，则执行步骤（4），否则执行步骤（5）。
  * 4.如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，即表示此对象处于轻量级锁定状态。
  * 5.如果这个更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行。否则说明多个线程竞争锁，轻量级锁就要膨胀为重量级锁，锁标志的状态值变为“10”，Mark Word中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。 而当前线程便尝试使用自旋来获取锁，自旋就是为了不让线程阻塞，而采用循环去获取锁的过程。
  
  偏向锁和轻量级锁的区别:
     引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令（由于一旦出现多线程竞争的情况就必须撤销偏向锁，所以偏向锁的撤销操作的性能损耗必须小于节省下来的CAS原子指令的性能消耗）。上面说过，轻量级锁是为了在线程交替执行同步块时提高性能，而偏向锁则是在只有一个线程执行同步块时进一步提高性能。


![Image text](http://7xq3rs.com2.z0.glb.qiniucdn.com/1535608753553x5TcMyCg_轻量级锁displaced.png)


![Image text](http://7xq3rs.com2.z0.glb.qiniucdn.com/1535608753505Nfad9n5J_轻量级锁指向.png)


## <span id="2.3">2.3 自旋锁</span>

自旋锁 可以通过 -XX:+UseSpinning  同时自旋的默认次数为10次，可以通过参数-XX:PreBlockSpin来调整； 

适应性自旋 自旋次数不固定

线程进入ContentionList之前，通过Linux下pthread_mutex_lock函数自旋




## <span id="2.4">2.4 重量级锁</span>
  指当锁为轻量级锁的时候，另一个线程虽然是自旋，但自旋不会一直持续下去，当自旋一定次数的时候，还没有获取到锁，就会进入阻塞，该锁膨胀为重量级锁。重量级锁会让其他申请的线程进入阻塞，性能降低。
  ![Image text](http://7xq3rs.com2.z0.glb.qiniucdn.com/15356087535970VteBzw4_重量级锁、轻量级锁和偏向锁之间转换.png)

## <span id="2.5">2.5 锁膨胀</span>
  偏向锁通过在Java对象的对象头markOop中install一个JavaThread指针的方式实现了这个Java对象对此Java线程的偏向，并且只有该偏向线程能够锁定Lock该对象。但是只要有第二个Java线程企图锁定这个已被偏向的对象时，偏向锁就不再满足这种情况了，然后呢JVM就将Biased Locking切换成了Basic Locking(基本对象锁)。Basic Locking使用CAS操作确保多个Java线程在此对象锁上互斥执行。如果CAS由于竞争而失败(第二个Java线程试图锁定一个正在被其他Java线程持有的对象)，这时基本对象锁因为不再满足需要从而JVM会切换到膨胀锁 - ObjectMonitor。
  
  是否会出现锁降级? 

  目的是为了提高获得锁和释放锁的效率。

    http://openjdk.java.net/jeps/8183909
    https://docs.oracle.com/cd/E13150_01/jrockit_jvm/jrockit/geninfo/diagnos/thread_basics.html

## <span id="2.6">2.6 锁降级</span>

   因为BasicLocking的实现优先于重量级锁的使用，JVM会尝试在STW的停顿中对处于“空闲(idle)”状态的重量级锁进行降级(deflate)。这个降级过程是如何实现的呢？我们知道在STW时，所有的Java线程都会暂停在“安全点(SafePoint)”，此时VMThread通过对所有Monitor的遍历，或者通过对所有依赖于MonitorInUseLists值的当前正在“使用”中的Monitor子序列进行遍历，从而得到哪些未被使用的“Monitor”作为降级对象。

   可降级的对象

  * 重量级锁的降级发生于STW阶段，降级对象就是那些仅仅能被VMThread访问而没有其他JavaThread访问的Monitor对象。

```java
void ObjectSynchronizer::deflate_idle_monitors() {
  assert(SafepointSynchronize::is_at_safepoint(), "must be at safepoint");
  int nInuse = 0 ;              // currently associated with objects
  int nInCirculation = 0 ;      // extant
  int nScavenged = 0 ;     

       // reclaimed
  bool deflated = false;

  ObjectMonitor * FreeHead = NULL ;  // Local SLL of scavenged monitors
  ObjectMonitor * FreeTail = NULL ;

  TEVENT (deflate_idle_monitors) ;
// Prevent omFlush from changing mids in Thread dtor's during deflation
// And in case the vm thread is acquiring a lock during a safepoint
// See e.g. 6320749
  Thread::muxAcquire (&ListLock, "scavenge - return") ;

  if (MonitorInUseLists) {
    int inUse = 0;
    for (JavaThread* cur = Threads::first(); cur != NULL; cur = cur->next()) {
      nInCirculation+= cur->omInUseCount;
      int deflatedcount = walk_monitor_list(cur->omInUseList_addr(), &FreeHead, &FreeTail);
      cur->omInUseCount-= deflatedcount;
// verifyInUse(cur);
      nScavenged += deflatedcount;
      nInuse += cur->omInUseCount;
     }

// For moribund threads, scan gOmInUseList
   if (gOmInUseList) {
     nInCirculation += gOmInUseCount;
     int deflatedcount = walk_monitor_list((ObjectMonitor **)&gOmInUseList, &FreeHead, &FreeTail);
     gOmInUseCount-= deflatedcount;
     nScavenged += deflatedcount;
     nInuse += gOmInUseCount;
    }

  } else for (ObjectMonitor* block = gBlockList; block != NULL; block = next(block)) {
// Iterate over all extant monitors - Scavenge all idle monitors.
    assert(block->object() == CHAINMARKER, "must be a block header");
    nInCirculation += _BLOCKSIZE ;
    for (int i = 1 ; i < _BLOCKSIZE; i++) {
      ObjectMonitor* mid = &block[i];
      oop obj = (oop) mid->object();

      if (obj == NULL) {
// The monitor is not associated with an object.
// The monitor should either be a thread-specific private
// free list or the global free list.
// obj == NULL IMPLIES mid->is_busy() == 0
        guarantee (!mid->is_busy(), "invariant") ;
        continue ;
      }
      deflated = deflate_monitor(mid, obj, &FreeHead, &FreeTail);

      if (deflated) {
        mid->FreeNext = NULL ;
        nScavenged ++ ;
      } else {
        nInuse ++;
      }
    }
  }

  MonitorFreeCount += nScavenged;

// Consider: audit gFreeList to ensure that MonitorFreeCount and list agree.

  if (ObjectMonitor::Knob_Verbose) {
    ::printf ("Deflate: InCirc=%d InUse=%d Scavenged=%d ForceMonitorScavenge=%d : pop=%d free=%d\n",
        nInCirculation, nInuse, nScavenged, ForceMonitorScavenge,
        MonitorPopulation, MonitorFreeCount) ;
    ::fflush(stdout) ;
  }

  ForceMonitorScavenge = 0;    // Reset

// Move the scavenged monitors back to the global free list.
  if (FreeHead != NULL) {
     guarantee (FreeTail != NULL && nScavenged > 0, "invariant") ;
     assert (FreeTail->FreeNext == NULL, "invariant") ;
// constant-time list splice - prepend scavenged segment to gFreeList
     FreeTail->FreeNext = gFreeList ;
     gFreeList = FreeHead ;
  }
  Thread::muxRelease (&ListLock) ;

  if (ObjectMonitor::_sync_Deflations != NULL) ObjectMonitor::_sync_Deflations->inc(nScavenged) ;
  if (ObjectMonitor::_sync_MonExtant  != NULL) ObjectMonitor::_sync_MonExtant ->set_value(nInCirculation);

// TODO: Add objectMonitor leak detection.
// Audit/inventory the objectMonitors -- make sure they're all accounted for.
  GVars.stwRandom = os::random() ;
  GVars.stwCycle ++ ;
}
```




## <span id="3">3. lock</span>

ReentrantLock
  AbstractQueuedSynchronizer实现
```java
 public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
  private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

LockSupport
    unpark(Thread thread)
    park(boolean isAbsolute,long time)

公平模式 非公平
```Java
public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```
  公平->避免饥饿，但是会有额外的维护开销，默认为非公平

 根据重入次数划分

  * 共享锁(Semaphore).可以被多个线程同时持有
  * 独占锁(ReentrantLock).一次只能被一个线程持有



ReentrantReadWriteLock 

  读读不加锁，读写，写写加锁 aqs高位读低位写

  ```java
   /*
         * Read vs write count extraction constants and functions.
         * Lock state is logically divided into two unsigned shorts:
         * The lower one representing the exclusive (writer) lock hold count,
         * and the upper the shared (reader) hold count.
         */

        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** Returns the number of shared holds represented in count  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** Returns the number of exclusive holds represented in count  */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
  ```

## <span id="4">4. 锁优化</span>

  * 减少锁持有时间
  * 减小锁粒度 ->ConcurrentHashMap
  * 锁分离
  * 锁粗化
  * 锁消除
 


# <span id="5">5. 乐观锁/悲观锁</span>
  在mysql数据库中的实现

  * 2pl
    - 锁操作两个阶段.
      + 加锁阶段
      + 解锁阶段
  *  mvcc 读不加锁，读写不冲突
    -  在MVCC并发控制中，读操作可以分成两类：快照读 (snapshot read)与当前读 (current read)。快照读，读取的是记录的可见版本 (有可能是历史版本)，不用加锁。当前读，读取的是记录的最新版本，并且，当前读返回的记录，都会加上锁，保证其他事务不会再并发修改这条记录。
    -  innodb rr级别 满足事务的隔离，通过版本号 select快照读 保证了读到的数据都是以提交的数据
    -    每开启一个事务，会生成一个事务版本号，被操作的数据会生成一条新的数据行，未提交直接对其他事务不可见。
    InnoDB数据行的结构
    最近修改该行的事务id | 回滚段指针     |  主键(如果未定义主键) |
    DATA_TRX_ID       | DATA_ROLL_PTR | PK                 | other columns

  CAS
    unsafe. compareAndSwapInt(Object o, long offset,int expected,int x);
    //src/os_cpu/linux_86/vm/auomic_linux_x86

//第一个参数o为给定对象，offset为对象内存的偏移量，通过这个偏移量迅速定位字段并设置或获取该字段的值，
//expected表示期望值，x表示要设置的值，
```java
  inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
    }

```
 Linux的X86下主要是通过cmpxchgl这个指令在CPU级完成CAS操作的，但在多处理器情况下必须使用lock指令加锁来完成。 

ABA问题
耗资源


 悲观锁

  * 表锁 开销小
  * 行锁 粒度小，开销大 
    * recordLock
    * gapLock
    * next key lock

# <span id="6">6.分布式锁</span>
  主要实现方式 

  * redis setNx
  * zookeeper 虚拟Node




#
![Image text](http://7xq3rs.com2.z0.glb.qiniucdn.com/1535608753649iz9ApUgQ_lock.png)




<!-- <meta http-equiv="refresh" content="2"> -->


