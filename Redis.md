# 数据类型

- 基本类型
  1. String：hello world
  2. Hash：{name: "Jack", sex: "1"}
  3. List：[A -> B -> C]
  4. Set：{A, B, C}
  5. SoredSet：{A:1, B:2, C:3}
- 特殊类型
  1. GEO：{A: (120.3, 30.5) }
  2. BitMap：01100111
  3. HyperLog：01100111



# 使用场景

- **缓存**
  - 穿透、击穿、雪崩
  - 双写一致、持久化
  - 数据过期、淘汰策略
- **分布式锁**
  - setnx、redisson
- 计数器
- 保存token
- 消息队列
- 延迟队列

### **集群**

- 主从
- 哨兵
- 集群

### **事务**

### Redis为什么快



# 缓存穿透

指访问一个缓存和数据库中都不存在的key，由于这个key在缓存中不存在，则会到数据库中查询，数据库中也不存在该key，无法将数据添加到缓存中，所以每次都会访问数据库导致数据库压力增大

解决方案：

1. 将空key添加到缓存中。
2. 使用布隆过滤器（bitmap位图）过滤空key。
3. 一般对于这种访问可能由于遭到攻击引起，可以对请求进行身份鉴权、数据合法行校验等。

# 缓存击穿

指大量请求访问缓存中的一个key时，该key过期了，导致这些请求都去直接访问数据库，短时间大量的请求可能会将数据库击垮。



解决方案：

1. 添加互斥锁或分布式锁，让一个线程去访问数据库，将数据添加到缓存中后，其他线程直接从缓存中获取。
2. 热点数据key不过期，定时更新缓存，但如果更新出问题会导致缓存中的数据一直为旧数据。
3. 逻辑过期，不设置redis过期时间，增加过期时间字段，通过逻辑判断加锁更新数据
![image-20230609011001535](https://github.com/710765989/learningDocument/blob/main/img/image-20230609011001535.png)

# 缓存雪崩

指在系统运行过程中，缓存服务宕机或大量的key值同时过期，导致所有请求都直接访问数据库导致数据库压力增大。



解决方案：

1. 将key的过期时间打散，避免大量key同时过期。
2. 对缓存服务做高可用处理。哨兵模式、集群模式
3. 加互斥锁，同一key值只允许一个线程去访问数据库，其余线程等待写入后直接从缓存中获取。
4. 添加降级限流策略
5. 添加多级缓存



# 双写一致性

- 延时一致：
  - 使用mq异步进行数据同步，更新数据之后通知删除
  - canal伪装成mysql从节点，读取binlog数据更新缓存
- 强一致，性能较低（Redisson）：
  - 共享锁：读锁ReadLock，其他线程的读操作不受影响，写操作阻塞
  - 排他锁：独占锁WriteLock，阻塞其他线程操作



# 持久化

## RDB（默认）

Redis Database Backup file

Redis创建快照，对快照进行备份

1. 本地持久化
2. 主从备份：将快照复制到其他服务器上就能创建相同数据的服务器



持久化配置

```clojure
save 900 1           #在900秒(15分钟)之后，如果至少有1个key发生变化，Redis就会自动触发bgsave命令创建快照。

save 300 10          #在300秒(5分钟)之后，如果至少有10个key发生变化，Redis就会自动触发bgsave命令创建快照。

save 60 10000        #在60秒(1分钟)之后，如果至少有10000个key发生变化，Redis就会自动触发bgsave命令创建快照。
```



###  RDB 创建快照时会阻塞主线程吗？

Redis 提供了两个命令来生成 RDB 快照文件：

- `save` : 主线程执行，会阻塞主线程；
- `bgsave` : 子线（进程？）程执行，不会阻塞主线程，默认选项。



## AOF（主流）

**append-only file**

与快照持久化相比，AOF 持久化的实时性更好

通过 appendonly 参数开启：

```bash
appendonly yes
```



每执行一条修改命令，Redis就会降命令写入`server.aof_buf`缓存中，然后根据下面的持久化配置决定刷盘时机

持久化配置

```bash
appendfsync always    #每次有数据修改发生时都会写入AOF文件,这样会严重降低Redis的速度
appendfsync everysec  #每秒钟同步一次，显式地将多个写命令同步到硬盘
appendfsync no        #让操作系统决定何时进行同步
```

`appendfsync everysec`让Redis每秒同步一次，性能几乎不受任何影响，如果系统崩溃，仅仅会损失1s的数据，而且Redis多用于缓存，丢失的数据重要程度较低。



```bash
bgrewriteaof # 重写AOF，用最少的命令达到相同的效果

# 自动重写配置
# AOF文件比上次文件 增长超过多少百分比则触发重写
auto-aof-rewrite-percentage 100
# AOF文件体积最小多大以上才触发重写
auto-aof-rewrite-min-size 64mb
```

![image-20230610012421668](https://github.com/710765989/learningDocument/blob/main/img/image-20230610012421668.png)



# 数据过期策略

- 懒删除
- 定期删除
  - SLOW：执行频率10hz，每次不超过25ms，可以通过修改配置文件`redis.conf`的hz选项进行调整
  - FAST：频率不固定，每次间隔不低于2ms，每次执行不超过1ms

redis结合这两种策略一起使用



# 数据淘汰策略

内存不够用时，进行添加新的数据，redis按照一定规则进行数据删除

1. noeviction：不允许写入新数据（**默认**）
2. volatile-ttl：对于设置了TTL（过期时间）的key，淘汰剩余过期时间较短的key
3. allkeys-random：对所有key，随机进行淘汰
4. volatile-random：对设置了TTL的key，随机进行淘汰
5. allkeys-lru：对所有key，基于**LRU（Least Recently Used最近最少使用：当前时间减去最近访问时间）**算法进行淘汰
6. volatile-lru：对设置了TTL的key，基于LRU算法进行淘汰
7. allkeys-lfu：对所有key，基于**LFU（Least Frequently Used最少频率使用：淘汰访问频率较低的key）**算法进行淘汰
8. volatile-lfu：对设置了TTL的key，基于LFU算法进行淘汰

![image-20230610014548994](https://github.com/710765989/learningDocument/blob/main/img/image-20230610014541840.png)



# 缓存读写策略

## Cache Aside Pattern（旁路缓存模式）

**写** ：

- 先更新 db
- 然后直接删除 cache 。

**读** :

- 从 cache 中读取数据，读取到就直接返回
- cache 中读取不到的话，就从 db 中读取数据返回
- 再把数据放到 cache 中。

![img](https://github.com/710765989/learningDocument/blob/main/img/cache-aside-write.png)



**Q&A**

1. **可以先删除 cache ，后更新 db 吗？**

   不可以。

   如果有两个请求A和B

   ​	A写入数据 -> A将缓存删除 -> B读取数据 -> B将读取到的**旧**数据写入缓存 -> A更新DB

   此时会导致DB和缓存中的数据不一致

2. **先更新DB，再删除缓存就没有问题吗？**

   理论上来讲，是有可能出现数据不一致的，不过由于缓存在内存中，读写速度是远远大于硬盘IO的，因此出现的概率可以忽略不计

## Read/Write Through Pattern（读写穿透）



## Write Behind Pattern（异步缓存写入）

# 分布式锁

![image-20230610213548066](https://github.com/710765989/learningDocument/blob/main/img/image-20230610213548066.png)

# 集群

## 全量同步

![image-20230610214323229](https://github.com/710765989/learningDocument/blob/main/img/image-20230610214323229.png)

## 增量同步

![image-20230610234734090](https://github.com/710765989/learningDocument/blob/main/img/image-20230610234734090.png)

![image-20230610234608994](https://github.com/710765989/learningDocument/blob/main/img/image-20230610234608994.png)
