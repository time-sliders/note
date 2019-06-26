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

Keidar and Dolev's E3PC[[2\]](https://en.wikipedia.org/wiki/Three-phase_commit_protocol#cite_note-E3PC-2) algorithm eliminates this disadvantage.

The protocol requires at least three round trips to complete, needing a minimum of three round trip times (RTTs). This is potentially a long latency to complete each transaction.

## References

1. Skeen, Dale; Stonebraker, M. (May 1983). "A Formal Model of Crash Recovery in a Distributed System". *IEEE Transactions on Software Engineering*. **9** (3): 219–228. [doi](https://en.wikipedia.org/wiki/Digital_object_identifier):[10.1109/TSE.1983.236608](https://doi.org/10.1109%2FTSE.1983.236608).
2. Keidar, Idit; Danny Dolev (December 1998). ["Increasing the Resilience of Distributed and Replicated Database Systems"](http://webee.technion.ac.il/~idish/Abstracts/jcss.html). *Journal of Computer and System Sciences (JCSS)*. **57** (3): 309–324. [doi](https://en.wikipedia.org/wiki/Digital_object_identifier):[10.1006/jcss.1998.1566](https://doi.org/10.1006%2Fjcss.1998.1566).

