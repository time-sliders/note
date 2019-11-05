三阶段提交协议在协调者和参与者中都引入超时机制，并且把两阶段提交协议的第一个阶段拆分成了两步：询问，然后再锁资源，最后真正提交。

**三个阶段的执行**
1. CanCommit阶段
3PC的CanCommit阶段其实和2PC的准备阶段很像。
协调者向参与者发送commit请求，参与者如果可以提交就返回Yes响应，否则返回No响应。

2.PreCommit阶段
Coordinator根据Cohort的反应情况来决定是否可以继续事务的PreCommit操作。
根据响应情况，有以下两种可能。
A.假如Coordinator从所有的Cohort获得的反馈都是Yes响应，那么就会进行事务的预执行：
发送预提交请求。Coordinator向Cohort发送PreCommit请求，并进入Prepared阶段。
事务预提交。Cohort接收到PreCommit请求后，会执行事务操作，并将undo和redo信息记录到事务日志中。
响应反馈。如果Cohort成功的执行了事务操作，则返回ACK响应，同时开始等待最终指令。

B.假如有任何一个Cohort向Coordinator发送了No响应，或者等待超时之后，Coordinator都没有接到Cohort的响应，那么就中断事务：
发送中断请求。Coordinator向所有Cohort发送abort请求。
中断事务。Cohort收到来自Coordinator的abort请求之后（**或超时之后，仍未收到Cohort的请求**），执行事务的中断。

3.DoCommit阶段

该阶段进行真正的事务提交，也可以分为以下两种情况:

执行提交

A.发送提交请求。Coordinator接收到Cohort发送的ACK响应，那么他将从预提交状态进入到提交状态。并向所有Cohort发送doCommit请求。
B.事务提交。Cohort接收到doCommit请求之后，执行正式的事务提交。并在完成事务提交之后释放所有事务资源。
C.响应反馈。事务提交完之后，向Coordinator发送ACK响应。
D.完成事务。Coordinator接收到所有Cohort的ACK响应之后，完成事务。

中断事务

Coordinator没有接收到Cohort发送的ACK响应（可能是接受者发送的不是ACK响应，**也可能响应超时**），那么就会执行中断事务。

**三阶段提交协议和两阶段提交协议的不同**

对于协调者(Coordinator)和参与者(Cohort)**都设置了超时机制**（在2PC中，只有协调者拥有超时机制，即如果在一定时间内没有收到cohort的消息则默认失败）。
在2PC的准备阶段和提交阶段之间，插入预提交阶段，使3PC拥有CanCommit、PreCommit、DoCommit三个阶段。
PreCommit是一个缓冲，保证了在最后提交阶段之前各参与节点的状态是一致的。

**优点**

三段提交协议的优点：能避免阻塞状态，在三段提交协议中，如果协调者在第二段之后失效，不会产生像2PC协议中可能出现的事务阻塞现象。因为下面两种状态至少存在一种：
1、所有参与者都进入Prepare to Commit状态，事务可以安全地提交。因为所有参与者都回答了ACK确认消息。
2、至少有一个参与者未进入Prepare to Commit状态，事务可以安全回滚。因为至少有一个参与者未回答ACK确认消息，则协调者也不会发出Globle-Commit命令。

三阶段提交协议(3PC)在恢复过程中协调者场地发生故障的情况下，也能够避免事务阻塞。这种方法的基本思想是在协调者发出prepare消息并收到所有下属的 yes 消息时，向所有下属发送 precommit 消息，而不是 commit 消息。然后在收到足够数目(比必须处理的最大故障数目要大)的ack消息以后，协调者向日志中强迫写入 commit 记录，再向所有下属发送 commit 消息。在3PC中，协调者场地能够有效地推迟commit决定，只有在确定所有下属都知道commit决定以后，才真正发出commit决定。如果协调者随后发生了故障，在协调者恢复之前，下属场地之间就可以互相通信，确定事务是应该提交还是中止事务(如果所有下属都没有收到precommit消息)。

**缺点**

三段提交协议缺点：虽能避免阻塞状态，但需要更多的通讯次数，实现比较复杂。因此实际应用较少。大多数使用一致性提交协议的系统都采用二阶段提交协议



总结：

三阶段提交相比于二阶段提交，不同点是，参与者引入超时机制，可以避免数据被锁定的时间过长，另外在相比二阶段提交

In [computer networking](https://en.wikipedia.org/wiki/Computer_networking) and [databases](https://en.wikipedia.org/wiki/Database), the **three-phase commit protocol** (**3PC**)[[1\]](https://en.wikipedia.org/wiki/Three-phase_commit_protocol#cite_note-3PC-1) is a [distributed algorithm](https://en.wikipedia.org/wiki/Distributed_algorithm) which lets all nodes in a [distributed system](https://en.wikipedia.org/wiki/Distributed_system)agree to [commit](https://en.wikipedia.org/wiki/Commit_(data_management)) a [transaction](https://en.wikipedia.org/wiki/Database_transaction). Unlike the [two-phase commit protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol) (2PC) however, 3PC is non-blocking. Specifically, 3PC places an upper bound on the amount of time required before a transaction either commits or [aborts](https://en.wikipedia.org/wiki/Abort_(computing)). This property ensures that if a given transaction is attempting to commit via 3PC and holds some [resource locks](https://en.wikipedia.org/wiki/Lock_(computer_science)), it will release the locks after the timeout.

## Protocol Description

In describing the protocol, we use terminology similar to that used in the [two-phase commit protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol). Thus we have a single coordinator site leading the transaction and a set of one or more cohorts being directed by the coordinator.

<img src='ref/Three-phase_commit_diagram.png' height='300px'>

### Coordinator

1. The coordinator receives a transaction request. If there is a failure at this point, the coordinator aborts the transaction (i.e. upon recovery, it will consider the transaction aborted). Otherwise, the coordinator sends a canCommit? message to the cohorts and moves to the waiting state.
2. If there is a failure, timeout, or if the coordinator receives a No message in the waiting state, the coordinator aborts the transaction and sends an abort message to all cohorts. Otherwise the coordinator will receive Yes messages from all cohorts within the time window, so it sends preCommit messages to all cohorts and moves to the prepared state.
3. If the coordinator succeeds in the prepared state, it will move to the commit state. However if the coordinator times out while waiting for an acknowledgement from a cohort, it will abort the transaction. In the case where an acknowledgement is received from the majority of cohorts, the coordinator moves to the commit state as well.

### Cohort

1. The cohort receives a canCommit? message from the coordinator. If the cohort agrees it sends a Yes message to the coordinator and moves to the prepared state. Otherwise it sends a No message and aborts. If there is a failure, it moves to the abort state.
2. In the prepared state, if the cohort receives an abort message from the coordinator, fails, or times out waiting for a commit, it aborts. If the cohort receives a preCommit message, it sends an **ACK** message back and awaits a final commit or abort.
3. If, after a cohort member receives a preCommit message, the coordinator fails or times out, the cohort member goes forward with the commit.

## Motivation

A [two-phase commit protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol) cannot dependably recover from a failure of both the coordinator and a cohort member during the **Commit phase**. If only the coordinator had failed, and no cohort members had received a commit message, it could safely be inferred that no commit had happened. If, however, both the coordinator and a cohort member failed, it is possible that the failed cohort member was the first to be notified, and had actually done the commit. Even if a new coordinator is selected, it cannot confidently proceed with the operation until it has received an agreement from all cohort members, and hence must block until all cohort members respond.

The three-phase commit protocol eliminates this problem by introducing the Prepared to commit state. If the coordinator fails before sending preCommit messages, the cohort will unanimously agree that the operation was aborted. The coordinator will not send out a doCommit message until all cohort members have **ACK**ed that they are **Prepared to commit**. This eliminates the possibility that any cohort member actually completed the transaction before all cohort members were aware of the decision to do so (an ambiguity that necessitated indefinite blocking in the [two-phase commit protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)).

## Disadvantages

The main disadvantage to this algorithm is that it cannot recover in the event the network is segmented in any manner. The original 3PC algorithm assumes a fail-stop model, where processes fail by crashing and crashes can be accurately detected, and does not work with network partitions or asynchronous communication.
该算法的主要缺点是在网络以任何方式分割的情况下都无法恢复。最初的3PC算法采用的是故障停止模型，在这种模型中，进程通过崩溃而失败，并且可以准确地检测到崩溃，并且不支持网络分区或异步通信。

Keidar and Dolev's E3PC[[2\]](https://en.wikipedia.org/wiki/Three-phase_commit_protocol#cite_note-E3PC-2) algorithm eliminates this disadvantage.

The protocol requires at least three round trips to complete, needing a minimum of three round trip times (RTTs). This is potentially a long latency to complete each transaction.

## References

1. Skeen, Dale; Stonebraker, M. (May 1983). "A Formal Model of Crash Recovery in a Distributed System". *IEEE Transactions on Software Engineering*. **9** (3): 219–228. [doi](https://en.wikipedia.org/wiki/Digital_object_identifier):[10.1109/TSE.1983.236608](https://doi.org/10.1109%2FTSE.1983.236608).
2. Keidar, Idit; Danny Dolev (December 1998). ["Increasing the Resilience of Distributed and Replicated Database Systems"](http://webee.technion.ac.il/~idish/Abstracts/jcss.html). *Journal of Computer and System Sciences (JCSS)*. **57** (3): 309–324. [doi](https://en.wikipedia.org/wiki/Digital_object_identifier):[10.1006/jcss.1998.1566](https://doi.org/10.1006%2Fjcss.1998.1566).

