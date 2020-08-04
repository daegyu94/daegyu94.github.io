---
title: "Distributed Commit"
date: 2020-08-04
categories: Distributed_Systems Distributed_Commit
---

# Distributed Commit

* Commit: making an operation permanent

## 2PC(Two-Phase Commit)

* Coordinator sends a VOTE_REQUEST

* Participant sends a VOTE_COMMIT or VOTE_ABORT

* Coordinator collects all votes and sends GLOBAL_COMMIT or GLOBAL_ABORT to all

* Processes commit or abort the transaction

* 2PC is consensus protocol

*  ![2PC](/assets/img/2PC.JPG)

* If participant is blocked in READY state, waiting for the global vote as sent by the coordinator,

  * Let a participant P contact another participant Q to see if it can decide from Q's current state

  * | State of Q | Action by P                 |
    | ---------- | :-------------------------- |
    | COMMIT     | Make transition to COMMIT   |
    | ABORT      | Make transition to ABORT    |
    | INIT       | Make transition to ABORT    |
    | READY      | Contact another participant |

  * This protocol can be blocked if the coordiantor crashed

### Coordinator Actions

* Record WAIT and then multicast VOTE_REQUEST to everyone
* After all decisions have been received, record the decision and then multicast

### Participant Actions

* Waits for a vote request
* Upon receiving a request, the participant decides the vote
* Records the vote and replies
* Logs the global decision and then executes
* DECISION_REQUEST if timeout (ask peer node)

### Steps taken by Coordinotor (2PC)

![Actions by coordinator](/assets/img/Actions by coordinator.JPG)

### Steps taken by Participant (2PC)

![Actions by participant](/assets/img/Actions by participant.JPG)

 ### Steps for handling incoming decision requests (2PC)

![Actions for handling decision requests](/assets/img/Actions for handling decision requests.JPG)

* Node  needs background thread to handle DECISION_REQUEST

### 2PC with Failures

* Participants fails in INIT
  * Coordinator will time out ==> GLOBAL_ABORT
* Participant fails in READY
  * After recovery, the participant should check the coordinator (wait for the decision)
    * Coordinator has not sent a decision yet ==> P will wait and get it
    * Coordinator already send a decision (COMMIT ==> COMMIT  or INIT/ABORT ==> ABORT) ==> P will get the decision from participants
* Coordinator fails in WAIT (got some votes from participants)
  * Select a new coordinator, start 2PC again for the transaction.
* Coordiantor fails after taking global decision
  * Scenario 1: One of the live participants received the decision
    * Other participants can ask for the decision to live participants ==> Nonblocking
  * Scenario 2: None of the live participatns received the decision.
    * One participan received the decision and committed, and then it also crashed.
      * Very rare case, but possible
    * Elect a new coordinator, and restart 2PC protocol
    * **Blocking**
      * Timeout <== Bounded waiting (not strictly blocking)
      * **restart 2PC <== Unbounded blocking** 
      * C, (P,Q,R) 이렇게 있을 때  C,P 가 P가 C로 부터 GLOBAL_COMMIT 을 받았다면, Q,R은 P가 복구 되어서 P가 GLOBAL_COMMIT을 받았는지 확인할 때까지 클러스터가 blocking 되어야한다.
    * Blocking is the greatest disadvantage of 2PC
    * In context of consensus requirements: 2PC is safe, but a blocking protocol 

## 3PC  (Three-Phase Commit)

* Goal: Turn 2PC into live (non-blocking) protocol
  * 3PC should never block on failures as 2PC does
* Insight: 2PC suffers from allowing nodes to irreversibly commit an outcome before ensuring that the others know the outcome, too. 
  * 분산된 노드가 COMMIT을 결정하면 다시 되돌릴 수가 없는 문제가 있다.
  * P, Q가 각자 GLOBAL_DECISION을 받았다는 걸 확인하고 COMMIT 한다.
* Idea in 3PC: split "commit/abort" phase into two phases
  * First communicate the outcome to everyone 
  * Let them commit only after everyone knows the outcome
  * READY 상태에서 바로 상태를 변경하지 말고 상태 하나를 더 추가하자.
* The states of the coordinator and each participant satisfy the following two conditions
  * **There is no single state from which it is possible to make a transition directly to either a COMMIT or an ABORT state**
  * **There is no state in which it is not possible to make a final decision, and from which a transition to COMMIT state can be made**
* ![3PC](/assets/img/3PC.JPG)
* PRECOMMIT 상태가 추가된다. Coordinator는 READY-COMMIT을 받으면 GLOBAL_COMMIT을 한다.

### 3PC Protocol

1. (Begin phase 1) Coordinator C sends Request-to-prepare to all participants (Vote mesg.)
2. Participants vote Prepared or No, just like 2PC
3. If C receives Prepared from all participants, then (begin phase 2) it sends Pre-Commit to all participants
4. Participants wait for Abort or Pre-Commit. Participant acknowledges(Ready-Commit) Pre-Commit
5. After C receives acks from all participants, or times out on some of them, it (begin third phase) sends Commit to all participants (that are up)

* 하나의 participant가 죽어도 다른 participant가 ready 상태이면 C가 COMMIT 보낸다.

### 3PC with Failures

* Coordinator fails after taking (PRECOMMIT) decision.
  * Scenario 2: None of the live participants received the decision
    * **One participant received the decision but it also failed (P가 PRECOMMIT 을 받았는데 fail...)**, P,Q 모두 PRECOMMIT을 못받은건 Ok ==> just GLOBAL_ABORT
    * **That decision is just PRECOMMIT. So it's ok since no actual commit occured.** (아직 값이 실제로 바뀌지 않았다.)
      * The coordinator will not send out a GLBOAL_Commit message until all participants have ACKed that they are in PRECOMMIT status.
      * This eliminates the possibility that any participant acutally compeleted the transaction before all participants were aware of the decision.

### Does 3PC achieve Consensus

* Liveness(availability)?
  * Yes, Doesn't block. It always makes progress by timing out
* Safety(correctness):
  * Yes for Synchronous systems
  * **Nope for Asynchronous system**
* When 3PC would result in inconsistent states between the replicas?
  * Two examples of unsafety in 3PC (Network partitions)
    * A participant hasn't crahsed, it's just offline
    * Coordinator hasn't crashed, it's just offline

### 3PC with Network Partitions

* One exampel scenario
  * Participant A receives PRECOMMIT from Coordinator
  * Then, A gets partitioned from B/C/D and Coordinator crashes
  * Noe of B/C/D have received PRECOMMIT, hence they all abort upon timeout
  * A is prepared to commit, hence, according to protocol, after it times out, it unilaterally decides to commit
    * Q: Why does it commit after it times out?
    * A: **Because it is a non-blocking protocol!**

### Safety vs Liveness

* So, 3PC is doomed for network partitions
  * The way to think about it is that this protocol's design trades safety for liveness
* Remember that 2PC traded liveness(but..blocking) for safety(consistency)
* 3PC is not safe but liveness
* Can we design a protocol that's both safe and live?
* Well, it turns out that it's impossible in the most general case!

### Fischer-Lynch-Paterson [FLP] impossibility Result

* It is impossible for a set of processorrs in an asynchronous system to agree on a binary value, even if only a single process is subject to an unannounced failure
* The core of the problem is asynchrony
  * It makes it impossible to tell whether or not a machine has crashed or you just can't reach it now
* For synchronous systems, 3PC guarantees both safety and liveness!
  * When you know the upper bound of message delays, you can infer when somthing has crashed with certainty

* But then, PAXOS is designed as completely-safe and largely-l;ived agrrement protocol
  * PAXOS lets all nodes agree on the same value despite node failures, network failures, and delays

### PAXOS Is Everywhere

* Cubby, Zookeeper, Frangipani, Scatter...
* Open source: libpaxos, Zookeeper



[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/