相关资料：https://redis.io/topics/persistence

# redis 持久化之 RDB & AOF

##### Redis 持久化实现方式

------

- 快照
  对数据某一时间点的完整备份。例如Linux 快照备份、Redis RDB、MySQL Dump。
- 日志
  将数据的所有操作都记录到日志中，需要恢复时，将日志重新执行一次。MySQL biglog、Redis AOF。

##### RDB

------

###### 什么是 RDB

将redis内存中的数据，完整的生成一个快照，以.rdb结尾的文件保存在硬盘上，当需要恢复时，再从文件加载到内存中。

###### RDB 三种触发方式

- save命令触发(同步)

```
[vagrant@tmwy ~]$ redis-cli
127.0.0.1:6379> save
OK
```

**save执行时，会造成Redis的阻塞。所有数据操作命令都要排队等待它完成**。
文件策略：新生成一个新的临时文件，当save执行完后，用新的替换老的。

- bgsave命令触发(异步)

```
[vagrant@tmwy ~]$ redis-cli
127.0.0.1:6379> bgsave
Background saving started
```

客户端对Redis服务器下达bgsave命令时，Redis会fork出一个子进程进行rdb文件的生成。当文件生成完毕后，子进程再反馈给主进程。fork子进程时也会阻塞，不过正常情况下fork过程都非常快的。
文件策略：与save命令相同。

- 配置文件配置规则自动触发

| 配置 | seconds | changes | 作用                                   |
| ---- | ------- | ------- | -------------------------------------- |
| save | 900     | 1       | 900秒内改变1条数据、自动生成rdb文件    |
| save | 300     | 10      | 300秒内改变10条数据、自动生成rdb文件   |
| save | 60      | 10000   | 60秒内改变10000条数据、自动生成rdb文件 |

*PS: 这三种规则都不建议使用。*

###### RDB 自动规则配置

------

```
# 配置自动生成规则。一般不建议配置自动生成rdb文件
save 900 1
save 300 10
save 60 10000
# 指定rdb文件名
dbfilename dump-${port}.rdb
# 指定rdb文件目录
dir /opt/redis/data
# bgsave发生错误，停止写入
stop-writes-on-bgsave-error yes
# rdb文件采用压缩格式
rdbcompression yes
# 对rdb文件进行校验
rdbchecksum yes
```

###### RDB 不容忽略的触发方式

------

- 全量复制
  **主从复制时，主会自动生成rdb文件**（主从就是依据rdb文件进行数据同步）。
- debug reload
  redis提供了debug级的重启，不清空内存的一种重启方式，也会生成rdb文件。
- shutdown
  **关闭redis会触发rdb文件生成。**

###### RDB 存在的问题

------

- **耗时、耗内存、耗IO性能**
  将内存中的数据全部dump到硬盘当中，耗时。bgsave的方式fork()子进程耗额外内存。大量的硬盘读写耗费IO性能。
- 不可控、丢失数据
  宕机时，上次快照之后写入的内存数据，将会丢失。

###### RDB 总结

------

- RDB是Redis内存到硬盘的快照，用于持久化。
- save通常会阻塞redis。
- bgsave通常不会阻塞redis，但是会fork新进程。
- save自动配置满足任一就会被执行。
- 耗时、耗内存、耗IO性能
- 不可控、丢失数据

##### AOF

------

###### 什么是 AOF

就是写日志，每次执行Redis写命令，让**命令同时记录日志**（以.aof结尾的日志文件）。Redis宕机时，只要进行**日志回放**就可以恢复数据。

###### AOF 三种策略

------

首先redis执行写命令将命令刷新到硬盘缓冲区中

- always
  总是让缓冲区文件刷新到硬盘(及时性)。
- everysec(推荐)
  每秒刷新一次缓冲区同步硬盘数据。
  对比always，在高写入量的情况下，可以保护硬盘。出故障时会丢失一秒数据
- no
  刷新策略让系统决定(不可控)。
- 三种策略对比

| 命令     | 优点         | 缺点                                               |
| -------- | ------------ | -------------------------------------------------- |
| always   | 不丢失数据   | IO开销大，一般的sata盘只有几百TPS                  |
| everysec | 只丢一秒数据 | 丢了一秒数据                                       |
| no       | 系统决定     | 不可控，不知道什么时候刷盘，也不知道会丢失多少数据 |

通常使用everysec策略，这也是AOF的默认策略。

###### AOF 重写

------

AOF重写就是把过期的、没用的、重复的以及可优化的命令，进行化简。只取最终有价值的结果。虽然写入操作很频繁，但系统定义的key的量是相对有限的。
AOF重写可以大大压缩最终日志文件的大小。从而减少磁盘占用量，加快数据恢复速度。比如我们有个计数的服务，有很多自增的操作，比如有一个key自增到1个亿，对AOF文件来说就是一亿次incr。AOF重写就只用记1条记录。

###### AOF 重写两种方式

------

- bgrewriteaof 命令触发AOF重写
  redis客户端向Redis发bgrewriteaof命令，redis服务端fork一个子进程去完成AOF重写。这里的AOF重写，是将Redis内存中的数据进行一次回溯，回溯成AOF文件。而不是重写AOF文件生成新的AOF文件去替换。
- AOF 重写配置
  - auto-aof-rewrite-min-size：AOF文件重写需要的尺寸
  - auto-aof-rewrite-percentage：AOF文件增长
  - aof_current_size：统计AOF当前尺寸(单位：字节)
  - aof_base_size：AOF上次启动和重写的尺寸(单位：字节)
- AOF自动重写的触发时机，需同时满足以下两点：
  - aof_current_size > auto-aof-rewrite-min-size
  - aof_current_size - aof_base_size/aof_base_size > auto-aof-rewrite-percentage

###### AOF 重写配置

------

```
# 开启正常AOF的append刷盘操作
appendonly yes
# AOF文件名
appendfilename "appendonly-${port}.aof"
# 每秒刷盘
appendfsync everysec
# 文件目录
dir /opt/redis/data
# AOF重写增长率
auto-aof-rewrite-percentage 100
# AOF重写最小尺寸
auto-aof-rewrite-min-size 64mb
# AOF重写期间是否暂停append操作。AOF重写非常消耗磁盘性能，而正常的AOF过程中也会往磁盘刷数据。
# 通常偏向考虑性能，设为yes。万一重写失败了，这期间正常AOF的数据会丢失，因为我们选择了重写期间放弃了正常AOF刷盘。
no-appendfsync-on-rewrite yes
```

##### RDB & AOF

------

###### RDB 对比 AOF

------

| 命令       | RDB    | AOF          | 说明                                                         |
| ---------- | ------ | ------------ | ------------------------------------------------------------ |
| 启动优先级 | 低     | 高           | RDB和AOF都开启的情况下，Redis重启后，选择AOF进行恢复。大部分情况下它保存了比RDB更新的数据 |
| 体积       | 小     | 大           | RDB二进制模式存储，而且做了压缩。AOF虽然有AOF重写，但是体积相对还是大很多，毕竟它是记日志形式 |
| 恢复速度   | 快     | 慢           | RDB体积小，恢复速度快。AOF体积大，恢复速度慢                 |
| 数据安全   | 丢数据 | 根据策略决定 | RDB丢上次快照后的数据，AOF根据always、everysec、no策略决定是否丢数据 |
| 轻重       | 重     | 轻           | AOF是追加日志，所以比较轻的操作。而RDB是CPU密集型操作，对磁盘，以及fork时对内存的消耗都比较大 |

###### RDB 最佳策略

------

- 建议关闭RDB
  无论是Redis主节点，还是从节点，都建议关掉RDB。但是关掉不是绝对的，主从复制时还是会借助RDB。
- 用作数据备份
  RDB虽然是很重的操作，但是对数据备份很有作用。文件大小比较小，可以按天或按小时进行数据备份。
- 主从，从开？
  在极个别的场景下，需要在从节点开RDB，可以再本地保存这样子的一个历史的RDB文件。虽然从节点不进行读写，但是Redis往往单机多部署，由于RDB是个很重的操作，所以还是会对CPU、硬盘和内存造成一定影响。根据实际需求进行设定。

###### AOF 最佳策略

------

- 建议开启AOF
  如果Redis数据只是用作数据源的缓存，并且缓存丢失后从数据源重新加载不会对数据源造成太大压力，这种情况下。AOF可以关。
- AOF重写集中管理
  单机多部署情况下，发生大量fork可能会内存爆满。
- everysec
  建议采用每秒刷盘策略

###### 最佳策略

------

- 小分片
  使用maxmemary对Redis最大内存进行规划。
- 缓存和存储
  根据缓存和存储的特性来决定使用哪种策略
- 监控（硬盘、内存、负载、网络）
- 足够的内存
  不要把就机器全部的内存规划给Redis。不然会出很多问题。像客户端缓冲区等，不受maxmemary限制。规划不当可能会产生SWAP、OOM等问题。

##### 开发运维常见问题

------

###### fork 操作

------

fork是一个同步操作。执行bgsave和bgrewriteaof时都会执行fork操作

- 改善fork
  - 优先使用物理机或者其他能高效支持form操作的虚拟化技术；
  - 控制Redis实例最大可用内存maxmemary；
    fork操作只是执行内存页的拷贝，大部分情况速度是比较快的。redis内存越大，内存页越大。可以使用maxmemary规划redis内存，避免fork过慢。
  - 合理配置Linux内存分配策略：vm.overcommit_memory=1
    fork时如果内存不够，会阻塞。Linux的vm.overcommit_memory默认为0，不会分配额外内存

###### 子进程开销和优化

------

bgsave和bgrewriteaof会进行fork操作产生子进程。

- CPU
  - 开销：RDB和AOF文件生成属于CPU密集型；
  - 优化：不做CPU绑定，不和CPU密集型应用部署在一起；
- 内存
  - 开销：fork内存开销
  - 优化：echo never > /sys/kernel/mm/transparent_hugepage/enabled
- 硬盘
  - 开销：AOF和RDB文件写入，可以结合iostat和iotao分析
  - 优化：
    - 不要和高硬盘负载服务部署在一起：存储服务、消息队列；
    - no-appendfsync-on-rewrite=yes；
    - 根据写入量决定磁盘类型：例如sdd；
    - 单机多实例持久化文件目录可以考虑分盘；

###### AOF 追加阻塞

------

**AOF阻塞定位**

- redis日志

```
Asynchronous AOF fsync is taking to long(disk is busy?). Writing the AOF 
buffer whitout waiting for fsync to complete, this may slow down Redis
```

- info persistence
  可以查看上述日志发生的次数:

```
127.0.0.1:6379> info persistence
......
......
aof_delayed_fsync: 100
......
......
```

**改善方式**

```
同子进程的硬盘优化
```