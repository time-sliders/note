# 1 Introduction 

The Paxos algorithm for implementing a fault-tolerant distributed system has been regarded as difficult to understand, perhaps because the original presentation was Greek to many readers [5]. In fact, it is among the simplest and most obvious of distributed algorithms. At its heart is a consensus algorithm—the “synod” algorithm of [5]. The next section shows that this consensus algorithm follows almost unavoidably from the properties we want it to satisfy. The last section explains the complete Paxos algorithm, which is obtained by the straightforward application of consensus to the state ma- chine approach for building a distributed system—an approach that should be well-known, since it is the subject of what is probably the most often-cited article on the theory of distributed systems [4]. 

# 2 The Consensus Algorithm 

## 2.1 The Problem 

Assume a collection of processes that can propose values. **A consensus algorithm ensures that a single one among the proposed values is chosen. If no value is proposed, then no value should be chosen. If a value has been chosen, then processes should be able to learn the chosen value.** The safety requirements for consensus are: 

* **Only a value that has been proposed may be chosen,** 
* **Only a single value is chosen, and** 
* **A process never learns that a value has been chosen unless it actually has been.** 

We won’t try to specify precise精确的 liveness活性 requirements. However, the goal is to ensure that some proposed value is eventually最终地 chosen and, if a value has been chosen, then a process can eventually learn the value. 

We let the three roles in the consensus algorithm be performed by three classes of agents: ***proposers***, ***acceptors***, and ***learners***. In an implementation, <u>a single process may act as more than one agen</u>t, but the mapping from agents to processes does not concern us here. 

Assume that agents can communicate with one another by sending messages. We use the **customary asynchronous, non-Byzantine model**非拜占庭模型, in which: 
* **Agents operate at arbitrary speed, may fail by stopping, and may restart. Since all agents may fail after a value is chosen and then restart, a solution is impossible unless some information can be remembered by an agent that has failed and restarted.** 
* **Messages can take arbitrarily long to be delivered送达, can be duplicated, and can be lost, but they are not corrupted损坏.** 

## 2.2 Choosing a Value 

The easiest way to choose a value is to have a single acceptor agent. A proposer sends a proposal to the acceptor, who chooses the first proposed value that it receives. Although虽然 simple, this solution is unsatisfactory不能令人满意的 because the failure of the acceptor makes any further progress impossible. 

So, let’s try another way of choosing a value. Instead of a single acceptor, let’s **use multiple acceptor agents**. A proposer sends a proposed value to a set of acceptors. An acceptor may accept the proposed value. **The value is chosen when a large enough set of acceptors have accepted it**. How large is large enough? To ensure that only a single value is chosen, **we can let a large enough set consist of any majority of the agents. Because any two majorities have at least one acceptor in common, this works if an acceptor can accept at most one value. 在一个acceptor最多接受一个元素的情况下这个方法是有效的** (There is an obvious显而易见 generalization结论 of a majority that has been observed评论 in numerous众多 papers, apparently starting with [3].) 

In the absence of在没有 failure or message loss, we want a value to be chosen even if only one value is proposed by a single proposer. This suggests the requirement: 

**$$P1$$.  An acceptor must accept the <u>first</u> proposal that it receives.**

But this requirement raises a problem. Several values could be proposed by different proposers at about the same time, leading to a situation in which every acceptor has accepted a value, but no single value is accepted by a majority of them. Even with just two proposed values, if each is accepted by about half the acceptors, failure of a single acceptor could make it impossible to learn which of the values was chosen. 

![Paxos-P1-Problem.png](ref/Paxos-P1-Problem.png)

P1 and the requirement that a value is chosen only when it is accepted by a majority of acceptors imply that **an acceptor must be allowed to accept more than one proposal**. We keep track of the different **proposals**that an acceptor may accept by **assigning a (natural自然数) number to each proposal**, so a proposal consists of **a proposal number** and **a value**. To prevent confusion不确定性, we require that <u>different **proposals** have different numbers</u>. How this is achieved depends on the implementation, so for now we just assume it. A value is chosen when a single proposal with that value has been accepted by a majority of the acceptors. In that case, we say that the proposal (as well as its value) has been chosen. 

> 由于每一个 acceptor 必须接受第一个 proposal，但是在最终，所有 acceptor 又必须要达到一致，所以一个 acceptor 必须要允许接受多个 proposal

We can allow multiple proposals to be chosen, but we must guarantee that all chosen proposals have the same value. By induction on the proposal number, it suffices to足以 guarantee: 

**$$P2$$. If a proposal with value $$v$$ is chosen, then every higher-numbered proposal that is chosen has value $$v$$.** **如果一个值为v的提案被选中, 那么更高编号的被选中的提案的值必须也为v** 

Since numbers are totally ordered全序的, condition P2 guarantees the crucial关键的 safety property that <u>**only a single value is chosen**</u>. 

To be chosen, a proposal must be accepted by at least one acceptor. So, we can satisfy满足 P2 by satisfying: 

**$$P2^a$$. If a proposal with value $$v$$ is chosen, then every higher-numbered proposal <u>accepted by any acceptor</u> has value $$v$$. ** **如果值为v的提案被选中, 那么后续任何acceptor能批准的更高版本的提案都必须要有值v.** 

We still maintain $$P1$$ to ensure that some proposal is chosen. Because communication is asynchronous, a proposal could be chosen with some particular acceptor $$c$$ never having received any proposal. Suppose a new proposer “wakes up” and issues a higher-numbered proposal with a different value. $$P1$$ requires $$c$$ to accept this proposal, violating $$P2^a$$. Maintaining both $$P1$$ and $$P2^a$$ requires strengthening  $$P2^a$$  to:  

由于通讯是异步的, 一个提案[m, v]在已经被选定之后可能还有某个acceptor c还没有接收到任何提议, 这个时候如果另一个proposer起草了一个版本更高的proposal: [m+k, v’], 而且如果[m+k, v’]是a收到的第一个提议, 根据P1, a就必须通过它. 这样P2a就很难保证了. 所以我们可以做一个更强的限定, 帮助我们使P2a成立: 

**$$P2^b$$. If a proposal with value $$v$$ is chosen, then every higher-numbered proposal <u>issued by any proposer</u> has value $$v$$.  ** **如果值为v的提案被选中, 那么后续任意的proposer所起草的更高编号的提案的值必须也是v.**

Since由于 a proposal must be issued by a proposer before it can be accepted by an acceptor, $$P2^b$$ implies意味着 $$P2^a$$, which in turn反过来 implies $$P2$$. 

To discover how to satisfy $$P2^b$$, let’s consider how we would prove that it holds让我们考虑一下如何证明这一点. We would assume that some proposal with number $$m$$ and value $$v$$ is chosen and show that any proposal issued with number $$n > m$$ also has value $$v$$. We would make the proof证明 easier by using induction on $$n$$, so we can prove that proposal number $$n$$ has value $$v$$ under the additional assumption that every proposal issued with a number in $$m . . (n − 1)$$ has value $$v$$ , where $$i . . j$$ denotes the set of numbers from $$i$$ through $$j$$ . For the proposal numbered $m$ to be chosen, there must be some set $C$ consisting of a majority of acceptors such that every acceptor in $C$ accepted it. Combining this with the induction assumption结合归纳假设, the hypothesis假设 that $m$ is chosen implies意味着: 

**Every acceptor in $C$ has accepted a proposal with number in  $m ..(n − 1)$, and every proposal with number in $m ..(n − 1)$  accepted by any acceptor has value $v$.**

Since any set $S$ consisting of a majority of acceptors contains at least one member of $C$, we can conclude推断出 that a proposal numbered $n$ has value $v$ by ensuring that the following invariant不变的 is maintained保持: 

**$P2^c$. For any $v$ and $n$, if a proposal with value $v$ and number $n$ is issued, then there is a set $S$ consisting of a majority of acceptors such that either  对于任意的N和V, 如果[N, V]被提出, 那么肯定存在一个由半数以上(majority)的acceptor组织的集合S, 满足下面两个条件之一
	(a) no acceptor in $S$ has accepted any proposal numbered less than $n$, or S中不存在任何批准过编号小于N的提案的Acceptor
	(b) $v$ is the value of the highest-numbered proposal among all proposals numbered less than $n$ accepted by the acceptors in $S$. S中所有Acceptor批准的编号小于N的提案中编号最大的值为V ** 

We can therefore satisfy $P2^b$ by maintaining the invariance of $P2^c$.
 To maintain the invariance of $P2^c$, a proposer that wants to issue a proposal numbered $n$ must learn the highest-numbered proposal with number less than $n$, if any, that has been or will be accepted by each acceptor in some majority of acceptors. Learning about proposals already accepted is easy enough; predicting future acceptances is hard. Instead of trying to predict the future, the proposer controls it by **extracting a promise索取承诺 that there won’t be any such acceptances认可，接受**. In other words, **the proposer requests要求 that the acceptors not accept any more proposals numbered less than n不再接受编号小于N的提案**. This leads to the following algorithm for issuing proposals. 

1. A proposer chooses a new proposal number $n$ and sends a request to each member of some set of acceptors, asking it to respond with: 

   (a) A promise never again to accept a proposal numbered less than $n$, and   承诺不会再通过任何比n小的提案

   (b) The proposal with the highest number less than $n$ that it has accepted, if any.  小于n的最大编号的提案(如有) 

   I will call such a request a ***prepare*** request with number $n$. 

2. If the proposer receives the requested responses from a **majority** of the acceptors, then it can issue a proposal with number $n$ and value $v$, where $v$ is the value of the highest-numbered proposal among the responses, or is any value selected by the proposer if the responders reported no proposals. 如果proposer受到了绝大多数acceptor的响应, 那么它就可以起草一个提案. 提案的编号是n, 值是根据 1 中的 (b) 里面响应的提案的值来的(如果没有提案, 或者提案值为空, 那么这个proposer可以用任意值) 

A proposer issues a proposal by sending, to some set of acceptors, a request that the proposal be accepted. (This need **not** be the **same** set of acceptors that responded to the initial requests.并不一定是上面prepare request发给的那个majority集合) Let’s call this an ***accept*** request. 

This describes a proposer’s algorithm. What about an acceptor? It can receive two kinds of requests from proposers: ***prepare*** requests and ***accept*** requests. An acceptor can ignore any request without compromising妥协 safety. So, we need to say only when it is allowed to respond to a request. It can always respond to a prepare request. It can respond to an accept request, accepting the proposal, **iff当且仅当 it has not promised not to**它总是可以响应准备请求。它可以响应接受请求，接受建议，如果它没有承诺不接受。. In other words: 

**$P1^a$ . An acceptor can accept a proposal numbered $n$ iff it has not responded to a prepare request having a number greater than $n$. **

Observe显然 that $P1^a$ subsumes子类 $P1$.
 We now have a complete algorithm for choosing a value that satisfies the required safety properties—assuming unique proposal numbers. The final algorithm is obtained by making one small optimization. 

Suppose an acceptor receives a prepare request numbered $n$, but it has already responded to a prepare request numbered greater than $n$, thereby promising not to accept any new proposal numbered $n$. There is then no reason for the acceptor to respond to the new prepare request 没有理由响应新的准备请求, since it will not accept the proposal numbered $n$ that the proposer wants to issue. **So we have the acceptor ignore such a prepare request. We also have it ignore a prepare request for a proposal it has already accepted.** 所以我们让接受者忽略这样一个准备请求。我们还让它忽略一个已经接受的提案的准备请求。

With this optimization, an acceptor needs to **remember only** the **highest numbered proposal that it has ever accepted** and **the number of the highest numbered prepare request to which it has responded**. Because $P2^c$ must be kept invariant不变 regardless无论 of failures, an **acceptor must remember this information even if it fails and then restarts**. Note that the proposer can always abandon a proposal and forget all about it—as long as只要 it never tries to issue another proposal with the same number. 

Putting the actions of the proposer and acceptor together, we see that the algorithm operates in the following two phases. 

**Phase 1.** 

**(a) A proposer selects a proposal number $n$ and sends a *prepare* request with number $n$ to a majority of acceptors.** 

**(b) If an acceptor receives a *prepare* request with number $n$ greater than that of any *prepare* request to which it has already responded, then it responds to the request with a promise not to accept any more proposals numbered less than $n$ and with the highest-numbered proposal (if any) that it has accepted.** 

**Phase 2.** 

**(a) If the proposer receives a response to its *prepare* requests (numbered $n$) from a majority of acceptors, then it sends an *accept* request to each of those acceptors for a proposal numbered $n$ with a value $v$, where $v$ is the value of the highest-numbered proposal among the responses, or is any value if the responses reported no proposals.** 

**(b) If an acceptor receives an *accept* request for a proposal numbered $n$, it accepts the proposal unless it has already responded to a *prepare* request having a number greater than $n$.** 

A proposer can make multiple proposals, so long as只要 it follows the algorithm for each one. It can abandon a proposal in the middle of the protocol at any time. (Correctness is maintained, even though requests and/or responses for the proposal may arrive at their destinations long after the proposal was abandoned.即使提案的请求和/或响应可能在提案被放弃之后的很长时间才到达目的地，正确性也能得到保证。) <u>It is probably a good idea to abandon a proposal if some proposer has begun trying to issue a higher-numbered one. Therefore, if an acceptor ignores a ***prepare*** or ***accept*** request because it has already received a ***prepare*** request with a higher number, then **it should probably inform the proposer, who should then abandon its proposal**. This is a performance optimization that does not affect correctness</u>.  				 			 		

![Consensus-Paxos.png](ref/Consensus-Paxos.png)

## 2.3 Learning a Chosen Value 

To learn that a value has been chosen, **<u>a learner must find out that a proposal has been accepted by a majority of acceptors</u>**. **The obvious algorithm is to have each acceptor, whenever it accepts a proposal, respond to all learners, sending them the proposal.** This allows learners to find out about a chosen value as soon as possible, but it requires each acceptor to respond to each learner—a number of responses equal to the product of the number of acceptors and the number of learners等于接受者数量和学习者数量的乘积. 

The assumption of non-Byzantine failures makes it easy for one learner to find out from another learner that a value has been accepted. We can have the acceptors respond with their acceptances to a distinguished杰出的 learner, which in turn informs the other learners when a value has been chosen我们可以让接受者对一个杰出的学习者作出接受的回应，当选择了一个值时，接受者又会通知其他学习者. This approach requires an extra round for all the learners to discover the chosen value这种方法需要额外的一轮学习，让所有的学习者发现所选的值. It is also less reliable, since the distinguished learner could fail因为杰出的学习者可能会失败. But it requires a number of responses equal only to the sum of the number of acceptors and the number of learners. 

More generally, **the acceptors could respond with their acceptances to some set of distinguished learners, each of which can then inform all the learners when a value has been chosen. ** Using a larger set of distinguished learners provides greater reliability at the cost of greater communication complexity. 使用一组更大的优秀学习者提供了更大的可靠性，同时也牺牲了更大的沟通复杂性。

Because of message loss, a value could be chosen with no learner ever finding out. The learner could ask the acceptors what proposals they have accepted, but failure of an acceptor could make it impossible to know whether or not a majority had accepted a particular proposal. In that case, learners will find out what value is chosen only when a new proposal is chosen. **If a learner needs to know whether a value has been chosen, it can have a proposer issue a proposal, using the algorithm described above.** 

## 2.4 Progress 

It’s easy to construct a scenario in which two proposers each keep issuing a sequence of proposals with increasing numbers, none of which are ever chosen. Proposer p completes phase 1 for a proposal number n1. Another proposer q then completes phase 1 for a proposal number n2 > n1. Proposer p’s phase 2 accept requests for a proposal numbered n1 are ignored because the acceptors have all promised not to accept any new proposal numbered less than n2. So, proposer p then begins and completes phase 1 for a new proposal number n3 > n2, causing the second phase 2 accept requests of proposer q to be ignored. And so on. 

To guarantee progress, **a distinguished杰出的 proposer** must be selected as the only one to try issuing proposals. If the distinguished proposer can communicate successfully with a majority of acceptors, and if it uses a proposal with number greater than any already used, then it will succeed in issuing a proposal that is accepted. By abandoning a proposal and trying again if it learns about some request with a higher proposal number, the distinguished proposer will eventually最终地 choose a high enough足够大的 proposal number. 

If enough of the system (proposer, acceptors, and communication network) is working properly, **liveness can therefore be achieved by electing a single distinguished proposer**那么可以通过选择一个杰出的提议者来实现活跃性. **The famous result of Fischer, Lynch, and Pat- terson [1] implies that a reliable algorithm for electing a proposer must use either randomness or real time—for example, by using timeouts. However, safety is ensured regardless of the success or failure of the election.** Fischer、Lynch和Pat-Terson[1]的著名结果表明，选择一个提议者的可靠算法必须使用随机性或实时性，例如，使用超时。然而，无论选举成功与否，都必须确保安全。

## 2.5 The Implementation 

The Paxos algorithm [5] assumes a network of processes. In its consensus algorithm, each process plays the role of proposer, acceptor, and learner. The algorithm chooses a **leader**, which plays the roles of the distinguished proposer and the distinguished learner. The Paxos consensus algorithm is precisely the one described above, where requests and responses are sent as ordinary messages. (Response messages are tagged with the corresponding proposal number to prevent confusion.) **Stable storage**, preserved during failures, is used to maintain the information that the acceptor must remember. An acceptor records its intended response in stable storage before actually sending the response. 

All that remains剩余的部分 is to **describe the mechanism for guaranteeing that no two proposals are ever issued with the same number. <u>Different proposers choose their numbers from disjoint不相交 sets of numbers</u>**, so two different proposers never issue a proposal with the same number. Each proposer remembers (in stable storage) the highest-numbered proposal it has tried to issue, and begins phase 1 with a higher proposal number than any it has already used. 

# 3 Implementing a State Machine

A simple way to implement a distributed system is as a collection of clients that issue commands to a central中心 server. The server can be described as a deterministic确定的 state machine that performs client commands in some sequence. The state machine has a current state; it performs a step by taking as input a command and producing an output and a new state. For example, the clients of a distributed banking system might be tellers出纳员, and the state-machine state might consist of the account balances of all users. A withdrawal取款 would be performed by executing a state machine command that decreases an account’s balance if and only if the balance is greater than the amount withdrawn, producing as output the old and new balances. 

An implementation that uses a single central server fails if that server fails. We therefore instead use a collection of servers, each one independently implementing the state machine. Because the state machine is deterministic, all the servers will produce the same sequences of states and outputs if they all execute the same sequence of commands. A client issuing a command can then use the output generated for it by any server. 

To guarantee that all servers execute the same sequence of state machine commands, we implement a sequence of separate instances of the Paxos consensus algorithm, the value chosen by the ith instance being the ith state machine command in the sequence. **Each server plays all the roles (proposer, acceptor, and learner) in each instance of the algorithm.** For now, I assume that **the set of servers is fixed**, so all instances of the consensus algorithm use the same sets of agents.

**In normal operation, a single server is elected to be the leader, which acts as the distinguished proposer (the only one that tries to issue proposals) in all instances of the consensus algorithm. Clients send commands to the leader, who decides where in the sequence each command should appear.** If the leader decides that a certain client command should be the 135th command, it tries to have that command chosen as the value of the 135th instance of the consensus algorithm. It will usually succeed. It might fail because of failures, or because another server also believes itself to be the leader and has a different idea of what the 135th command should be. But the consensus algorithm ensures that at most one command can be chosen as the 135th one. 

Key to the efficiency of this approach is that, in the Paxos consensus algorithm, the value to be proposed is not chosen until phase 2. Recall that, after completing phase 1 of the proposer’s algorithm, either the value to be proposed is determined or else the proposer is free to propose any value. 

I will now describe how the Paxos state machine implementation works during normal operation. Later, I will discuss what can go wrong. I consider what happens when the previous leader has just failed and a new leader has been selected. (System startup is a special case in which no commands have yet been proposed.) 

The new leader, being a learner in all instances of the consensus algo- rithm, should know most of the commands that have already been chosen. Suppose it knows commands 1–134, 138, and 139—that is, the values cho- sen in instances 1–134, 138, and 139 of the consensus algorithm. (We will see later how such a gap in the command sequence could arise.) It then executes phase 1 of instances 135–137 and of all instances greater than 139. (I describe below how this is done.) Suppose that the outcome of these ex- ecutions determine the value to be proposed in instances 135 and 140, but leaves the proposed value unconstrained in all other instances. The leader then executes phase 2 for instances 135 and 140, thereby choosing commands 135 and 140. 

The leader, as well as any other server that learns all the commands the leader knows, can now execute commands 1–135. However, it can’t execute commands 138–140, which it also knows, because commands 136 and 137 have yet to be chosen. The leader could take the next two commands requested by clients to be commands 136 and 137. Instead, we let it fill the gap immediately by proposing, as commands 136 and 137, a special “no- op” command that leaves the state unchanged. (It does this by executing phase 2 of instances 136 and 137 of the consensus algorithm.) Once these no-op commands have been chosen, commands 138–140 can be executed. Commands 1–140 have now been chosen. The leader has also completed phase 1 for all instances greater than 140 of the consensus algorithm, and it is free to propose any value in phase 2 of those instances. It assigns command number 141 to the next command requested by a client, proposing it as the value in phase 2 of instance 141 of the consensus algorithm. It proposes the 

next client command it receives as command 142, and so on.
 The leader can propose command 142 before it learns that its proposed command 141 has been chosen. It’s possible for all the messages it sent in proposing command 141 to be lost, and for command 142 to be chosen before any other server has learned what the leader proposed as command 141. When the leader fails to receive the expected response to its phase 2 messages in instance 141, it will retransmit those messages. If all goes well, its proposed command will be chosen. However, it could fail first, leaving a gap in the sequence of chosen commands. In general, suppose a leader can get α commands ahead—that is, it can propose commands i + 1 through i +α after commands 1 through i are chosen. A gap of up to α−1 commands 

could then arise.
 A newly chosen leader executes phase 1 for infinitely many instances 

of the consensus algorithm—in the scenario above, for instances 135–137 and all instances greater than 139. Using the same proposal number for all instances, it can do this by sending a single reasonably short message to the other servers. In phase 1, an acceptor responds with more than a simple OK only if it has already received a phase 2 message from some proposer. (In the scenario, this was the case only for instances 135 and 140.) Thus, a server (acting as acceptor) can respond for all instances with a single reasonably short message. Executing these infinitely many instances of phase 1 therefore poses no problem. 

Since failure of the leader and election of a new one should be rare events, the effective cost of executing a state machine command—that is, of achieving consensus on the command/value—is the cost of executing only phase 2 of the consensus algorithm. It can be shown that phase 2 of the Paxos consensus algorithm has the minimum possible cost of any algorithm for reaching agreement in the presence of faults [2]. Hence, the Paxos algorithm is essentially optimal. 

This discussion of the normal operation of the system assumes that there is always a single leader, except for a brief period between the failure of the current leader and the election of a new one. In abnormal circumstances, the leader election might fail. If no server is acting as leader, then no new commands will be proposed. If multiple servers think they are leaders, then they can all propose values in the same instance of the consensus algo- rithm, which could prevent any value from being chosen. However, safety is preserved—two different servers will never disagree on the value chosen as the ith state machine command. Election of a single leader is needed only to ensure progress. 

If the set of servers can change, then there must be some way of deter- mining what servers implement what instances of the consensus algorithm. The easiest way to do this is through the state machine itself. The current set of servers can be made part of the state and can be changed with ordi- nary state-machine commands. We can allow a leader to get α commands ahead by letting the set of servers that execute instance i + α of the con- sensus algorithm be specified by the state after execution of the ith state machine command. This permits a simple implementation of an arbitrarily sophisticated reconfiguration algorithm. 

References 

1. [1]  Michael J. Fischer, Nancy Lynch, and Michael S. Paterson. Impossibility of distributed consensus with one faulty process. Journal of the ACM, 32(2):374–382, April 1985. 
2. [2]  Idit Keidar and Sergio Rajsbaum. On the cost of fault-tolerant consensus when there are no faults—a tutorial. TechnicalReport MIT-LCS-TR-821, Laboratory for Computer Science, Massachusetts Institute Technology, Cambridge, MA, 02139, May 2001. also published in SIGACT News 32(2) (June 2001). 
3. [3]  Leslie Lamport. The implementation of reliable distributed multiprocess systems. Computer Networks, 2:95–114, 1978. 
4. [4]  Leslie Lamport. Time, clocks, and the ordering of events in a distributed system. Communications of the ACM, 21(7):558–565, July 1978. 
5. [5]  Leslie Lamport. The part-time parliament. ACM Transactions on Computer Systems, 16(2):133–169, May 1998. 