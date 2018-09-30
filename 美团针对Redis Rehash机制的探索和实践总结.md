redis

gitchat:https://gitbook.cn/books/5b62db7ea120e74da6655c43/index.html
mp:https://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651748475&idx=1&sn=ce1ea79eebf621b9a9365f15b5bab40a&chksm=bd12a1368a6528209fb2b520d79ac95c4bed4514dce92cde68b67fa862fcc9797f5cbd30c423&mpshare=1&scene=1&srcid=0905dNMCCJyQqmfHuo1lr18W#rd

1. 美团的Redis 是采用社区原生的Cluster模式，还是自研？如果自研主要做了哪些方面的改进？
    Squirrel；
2. 针对Redis Rehash 扩展和缩容这块美团如何规避和优化的？
    (1) slave 不触发被动驱逐淘汰(避免主从不一致)，Rehash时，根据内存使用，判断是否进行Resize：
    (2) 提前监控运营，在容量预估时将Rehash的内存占用也考虑在内，或运维中做好监控和告警，提前进行扩容等操作。
    主要原理是：通过Dict的二进制掩码，从高位不断加1，向低位推进的方式来实现全量扫描以及在Resize过程，因为Resize导致Dict 桶的位置发生迁移的场景。可以做到尽可能少重复返回。
    我们梳理源码和原理后，发现可以在大表中也通过高位访问来避免这问题，并且push到了redis源码社区，已经被官方merge。
    Rehash下的Scan 这块，主要的关注点有以下：
    (1) Dict是2^n个，且Resize也是按照2^n来进行，这块是实现高位访问的根本；
    (2) 集合 Dict掩码来实时计算游标
3. Redis Scan主要的原理以及Rehash 缩容对其影响的原理是？
    Redis的Scan的原理功能主要就是针对Rehash的，因为在Scan过程中，如果发生Rehash那么桶和Key都会动态移动，在这个过程中，作者为了避免过多重复的返回数据或遗漏数据，采用了一种称为 反向二进制和大小表游标结合的算法。

4. 在使用redis时，如何避免rehash呢？（@Verlin）
5. 针对缓存穿透，雪崩和热点key问题，Squirrel是如何应对的？（@小红帽）
     缓存雪崩，就是避免大量并发请求抵达缓存后端数据源，这块可以通过主动更新缓存。
     还有可以错开过期周期，避免大量集中。
     缓存穿透和热点Key 可以结合讲，比如某些Key 并发量很大或者Key不存在，导致Key在缓存层无法命中，然后再抵达后端数据源，这块，可以通过将热Key打散，以及缓存为空或者布隆过滤来避免后端数据源的崩溃。

ApsaraDB ali

会有的 比如底层 numa 多队列网卡 内存分配，还有基于Redis本身的内部机制和改进的。