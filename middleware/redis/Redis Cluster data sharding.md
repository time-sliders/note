原文地址：[cluster-tutorial](https://redis.io/topics/cluster-tutorial)

## Redis Cluster data sharding -redis集群数据分片

Redis Cluster does not use **consistent hashing**, but a different form of sharding where every key is conceptually part of what we call an **hash slot**.
redis 集群模式并没有使用一致性哈希，而是采用了一种不同的分片形式：每一个key属于我们“hash slot”概念的一部分

There are 16384 hash slots in Redis Cluster, and to compute what is the hash slot of a given key, we simply take the CRC16 of the key modulo 16384.
redis集群有16384个hash slot，根据一个指定的key可以计算出在哪个hash slot，我们只简单的对key进行CRC16校验（Cyclic Redundancy Check CRC即循环冗余校验码）

Every node in a Redis Cluster is responsible for a subset of the hash slots, so for example you may have a cluster with 3 nodes, where:
每一个redis集群的node都负责hash slot 的一部分，比如你可能有一个包含三个节点的集群

- Node A contains hash slots from 0 to 5500. 
- Node B contains hash slots from 5501 to 11000.
- Node C contains hash slots from 11001 to 16383.

This allows to **add and remove nodes in the cluster easily**. For example if I want to add a new node D, I need to move some hash slot from nodes A, B, C to D. Similarly if I want to remove node A from the cluster I can just move the hash slots served by A to B and C. When the node A will be empty I can remove it from the cluster completely.
这允许你很方便的在集群中添加和删除节点，比如我们想要新增一个节点D，我们需要将A,B,C 的一些hash slot移给D，类似的如果我想把A从集群中移走，我可以把A负责的节点移给B、C，当节点A空掉的时候，我们就可以把A完全地从集群中移除。

Because moving hash slots from a node to another **does not require to stop operations**, adding and removing nodes, or changing the percentage of hash slots hold by nodes, does **not require any downtime**.
由于将一个hash slots从一个节点移动到另外一个节点不需要进行 stop 操作，增加、删除节点，或者变更一个node节点持有的hash slots百分比并不需要任何停机时间。

Redis Cluster supports multiple key operations as long as all the keys involved into a single command execution (or whole transaction, or Lua script execution) all belong to the same hash slot. The user can force multiple keys to be part of the same hash slot by using a concept called ***hash tags***.
redis 集群支持多个key的操作，只要在一个执行指令的所有的keys全部属于同一个 hash slot。用户可以使用“hash tags”把强制多个key分到同一个hash slot

Hash tags are documented in the Redis Cluster specification, but the gist is that if there is a substring between {} brackets in a key, only what is inside the string is hashed, so for example `this{foo}key` and `another{foo}key` are guaranteed to be in the same hash slot, and can be used together in a command with multiple keys as arguments.
在 redis 集群手册中对 hash tags 已经说明过了，但是要点是，如果在一个key中有一个{}子字符串，只有括号内部的String会被hash处理，比如 `this{foo}key` 和 `another{foo}key` 可以保证在一个相同的hash slot 里面，所以我们可以作为一个多个参数命令的keys来一起使用



## Redis Cluster master-slave model

In order to remain available when a subset of master nodes are failing or are not able to communicate with the majority of nodes, Redis Cluster uses a master-slave model where every hash slot has from 1 (the master itself) to N replicas (N-1 additional slaves nodes).
为了在部分master nodes节点失败或者无法与集群中大部分节点通讯的时候保持可用性，redis集群使用 主从模式，每一个hash slot都有1-n个副本。

In our example cluster with nodes A, B, C, if node B fails the cluster is not able to continue, since we no longer have a way to serve hash slots in the range 5501-11000.
在我们的例子中集群有ABC三个节点，如果节点B失败，这个集群将无法继续服务，因为我们不在有办法去服务5501-11000范围类的hash slot。

However when the cluster is created (or at a later time) we add a slave node to every master, so that the final cluster is composed of A, B, C that are masters nodes, and A1, B1, C1 that are slaves nodes, the system is able to continue if node B fails.
然而当我们为每一个集群节点都添加一个slave节点的时候，集群由ABC三个 master node节点 和 A1,B1,C1三个从节点组成，如果节点B失败系统依然可以继续服务

Node B1 replicates B, and B fails, the cluster will promote node B1 as the new master and will continue to operate correctly.
节点B1是B的副本，如果B失败，集群会将B1升级为一个新的master，然后继续正确的运行。

However note that if nodes B and B1 fail at the same time Redis Cluster is not able to continue to operate.
然而需要注意如果 B 与 B1在同一时间失败，redis集群将没办法继续运行。


## Redis Cluster consistency guarantees- redis 集群一致性保证

Redis Cluster is not able to guarantee **strong consistency**. In practical terms this means that under certain conditions it is possible that Redis Cluster will lose writes that were acknowledged by the system to the client.
redis 集群没办法保证 "**强一致性**" 在实际上，这意味着在特定的条件下可能出现redis集群丢失写那些系统已经告诉客户端写入成功的数据

The first reason why Redis Cluster can lose writes is because it uses asynchronous replication. This means that during writes the following happens:
redis集群会丢失数据的第一个原因是因为采用了异步副本机制，这意味着在按照如下方式写数据时：

- Your client writes to the master B.
- The master B replies OK to your client.
- The master B propagates the write to its slaves B1, B2 and B3.

As you can see B does not wait for an acknowledge from B1, B2, B3 before replying to the client, since this would be a prohibitive latency penalty for Redis, so if your client writes something, B acknowledges the write, but crashes before being able to send the write to its slaves, one of the slaves (that did not receive the write) can be promoted to master, losing the write forever.
你可以看到B不会等待B1，B2，B3的返回之后才告诉客户端，因为这可能成为一个redis的静止延迟惩罚，所以如果客户端写一些数据，B告知写操作完成，并且在发送数据给slave之前宕机，某一个slave会被升级为 master，永远丢失这个写入的数据。

This is **very similar to what happens** with most databases that are configured to flush data to disk every second, so it is a scenario you are already able to reason about because of past experiences with traditional database systems not involving distributed systems. Similarly you can improve consistency by forcing the database to flush data on disk before replying to the client, but this usually results into prohibitively low performance. That would be the equivalent of synchronous replication in the case of Redis Cluster.
这与大多数数据库配置每秒刷数据到磁盘发生的现象很相似，因此这是一个你已经能够解释的场景，因为过去传统数据库的经验（不包含分布式系统）。类似的你可以通过强制数据库在返回结果给客户端之前，刷数据到磁盘来保证一致性，但是这通常会导致难以承受的低性能。在redis集群的情况下，这相当于同步复制。

Basically there is a trade-off to take between performance and consistency.
基本上这里有一个在性能和一致性上的权衡。

Redis Cluster has support for synchronous writes when absolutely needed, implemented via the [WAIT](https://redis.io/commands/wait) command, this makes losing writes a lot less likely, however note that Redis Cluster does not implement strong consistency even when synchronous replication is used: it is always possible under more complex failure scenarios that a slave that was not able to receive the write is elected as master.
当必须需要的时候，redis集群已经支持同步写，通过 WAIT 命令来实现，这会让写丢失更少的出现，然而注意redis集群不支持强一致性，即使使用了同步备份模式：在更复杂的失败场景下总有可能出现，比如一个无法接受这个write的slave被选为了master。

There is another notable scenario where Redis Cluster will lose writes, that happens during a network partition where a client is isolated with a minority of instances including at least a master.
还有一个会导致redis集群丢失写的场景，在网络分片的时候一个客户端与大多数隔离（至少包括master）

Take as an example our 6 nodes cluster composed of A, B, C, A1, B1, C1, with 3 masters and 3 slaves. There is also a client, that we will call Z1.
举个例子，我们有一个由 A,B,C,A1,B1,C1 6个节点组成的3master 3slave的集群，然后一个客户端节点，我们称为Z1

After a partition occurs, it is possible that in one side of the partition we have A, C, A1, B1, C1, and in the other side we have B and Z1.
当网络分片发生的时候，可能出现 A,  C, A1, B1, C1 与 B，Z1被分为网络的2部分。

Z1 is still able to write to B, that will accept its writes. If the partition heals in a very short time, the cluster will continue normally. However if the partition lasts enough time for B1 to be promoted to master in the majority side of the partition, the writes that Z1 is sending to B will be lost.
Z1依然可以写数据给B，并且B可以接受写操作，如果分片在很快恢复，集群会继续正常工作，然而如果分片时间足够长，B1在多数机器的分片部分被升级为master，Z1发送给B的写操作将会被丢失。

Note that there is a **maximum window** to the amount of writes Z1 will be able to send to B: if enough time has elapsed for the majority side of the partition to elect a slave as master, every master node in the minority side stops accepting writes.
注意这里Z1能发给B的数量有一个“最大窗口""：如果足够大多数机器重新选出一个master的时间过去了，每一个少部分集群的旧maste节点将停止接收写请求。

This amount of time is a very important configuration directive of Redis Cluster, and is called the **node timeout**.
时间的量是redis集群的一个非常重要的直接配置，这里称为"node-timeout"

After node timeout has elapsed, a master node is considered to be failing, and can be replaced by one of its replicas. Similarly after node timeout has elapsed without a master node to be able to sense the majority of the other master nodes, it enters an error state and stops accepting writes.
如过node的timeout时间到达，一个mater节点将会被认为失败，可以被它的一个副本所替换。类似在一个节点的timeout时间到达的时候，如果一个master节点还没有感知大部分其他master节点，master将会进入一个错误的状态，并且不在接收写请求。