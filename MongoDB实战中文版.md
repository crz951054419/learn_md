1.数据备份
./mongodump -data mongo_test -o mongo_dump
2.数据恢复
./mongorestore -d mongo_test mongo_dump/*
3.绑定ip 内网访问mongo服务
./mongod --bind_ip 120.0.0.1
4.查看活动进程
db.currentOp();

5.结束进程

db.killOp(opId);

6.索引

db.user.ensureIndex({ age:1}) 1->升序 -1降序

后台创建,{background:true}

查看索引情况

db.user.getIndexes();

唯一索引

db.user.ensureIndex({age:1},{unique:true})

执行计划

db.hit_quiz_record.find().explain();