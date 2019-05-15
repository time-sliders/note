原文地址： <https://redis.io/topics/distlock>

This is unfortunately not viable.  
不幸的是，这是不可行的

Fault tolerance. As long as the majority of Redis nodes are up, clients are able to acquire and release locks.  
容错。只要大多数redis节点都启动，客户端就可以获取和释放锁

Deadlock free. Eventually it is always possible to acquire a lock, even if the client that locked a resource crashes or gets partitioned.  
无死锁。最终，即使锁定资源的客户机崩溃或分区，也始终可以获取锁。

Why failover-based implementations are not enough  
为什么基于故障转移的实现不够

There is an obvious race condition with this model  
这个模型有一个明显的竞争条件

Those nodes are totally independent, so we don’t use replication or any other implicit coordination system  
这些节点是完全独立的，因此我们不使用复制或任何其他隐式协调系统。

the client uses a timeout which is small compared to the total lock auto-release time in order to acquire it.  
客户机使用一个比锁自动释放总时间小的超时来获取它。

The algorithm relies on the assumption that while there is no synchronized clock across the processes, still the local time in every process flows approximately at the same rate, with an error which is small compared to the auto-release time of the lock. This assumption closely resembles a real-world computer: every computer has a local clock and we can usually rely on different computers to have a clock drift which is small.   
该算法基于这样一个假设，即虽然进程之间没有同步时钟，但每个进程中的本地时间仍以大致相同的速率流动，与锁的自动释放时间相比，误差较小。这一假设与现实世界中的计算机非常相似：每台计算机都有一个本地时钟，我们通常可以依靠不同的计算机来产生一个小的时钟漂移。

an availability penalty to pay as it waits for key expiration  
等待密钥到期时支付的可用性惩罚

When a client is unable to acquire the lock, it should try again after a random delay in order to try to desynchronize multiple clients trying to acquire the lock for the same resource at the same time (this may result in a split brain condition where nobody wins). Also the faster a client tries to acquire the lock in the majority of Redis instances, the smaller the window for a split brain condition (and the need for a retry), so ideally the client should try to send the SET commands to the N instances at the same time using multiplexing.  

It is worth stressing how important it is for clients that fail to acquire the majority of locks, to release the (partially) acquired locks ASAP, so that there is no need to wait for key expiry in order for the lock to be acquired again (however if a network partition happens and the client is no longer able to communicate with the Redis instances, there is an availability penalty to pay as it waits for key expiration).

当客户机无法获取锁时，应在随机延迟后重试，以尝试使试图同时获取同一资源的锁的多个客户机不同步（这可能导致没有人获胜的分裂大脑状况）。而且，在大多数redis实例中，客户端试图获取锁的速度越快，分裂大脑状态的窗口越小（并且需要重试），因此理想情况下，客户端应尝试使用多路复用将set命令同时发送到n个实例。

值得强调的是，对于无法获取大多数锁的客户机来说，尽快释放（部分）获取的锁是多么重要，这样就不需要等待密钥到期，以便再次获取锁（但是，如果发生网络分区，并且客户机不再能够与Redis实例通信，在等待密钥到期时，需要支付可用性惩罚）。

Are you able to provide a formal proof of safety, point to existing algorithms that are similar, or find a bug? That would be greatly appreciated.  
您是否能够提供正式的安全证明，指向与之类似的现有算法，或者发现错误？那将非常感谢。

If the work performed by clients is composed of small steps, it is possible to use smaller lock validity times by default, and extend the algorithm implementing a lock extension mechanism. Basically the client, if in the middle of the computation while the lock validity is approaching a low value, may extend the lock by sending a Lua script to all the instances that extends the TTL of the key if the key exists and its value is still the random value the client assigned when the lock was acquired.  

The client should only consider the lock re-acquired if it was able to extend the lock into the majority of instances, and within the validity time (basically the algorithm to use is very similar to the one used when acquiring the lock).

However this does not technically change the algorithm, so the maximum number of lock reacquisition attempts should be limited, otherwise one of the liveness properties is violated.

如果客户端执行的工作是由小步骤组成的，则可以默认使用较小的锁有效期，并扩展实现锁扩展机制的算法。基本上，如果在计算过程中，当锁的有效性接近一个较低的值时，客户机可以通过向所有实例发送一个lua脚本来扩展锁，如果该键存在，并且其值仍然是客户机在获取锁时分配的随机值，则可以扩展该锁。

客户机应该只考虑重新获取的锁，如果它能够将锁扩展到大多数实例中，并且在有效期内（基本上使用的算法与获取锁时使用的算法非常相似）。

但是，这并不能从技术上改变算法，因此应该限制锁重新获取的最大尝试次数，否则将违反其中一个活动属性。
