# Raft algorithm
## Papers
### In Search of an Understandable Consensus Algorithm (Extended Version)
#### By: Diego Ongaro and John Ousterhout Stanford University
##### Section1:Raft features
- Strong leader
  - the log flows only from leader to other servers
- Leader election
  - user random timers to elect leaders
  - (?) This adds only a small amount of mechanism to the heartbeats already required for any consensus algorithm, while resolving conﬂicts simply and rapidly.
  - By searching, it means that Raft uses random timers to start electing, which will simply seperate the votes so that it won't have conﬂicts during election.
- membership changes
  - Raft uses ***joint consensus*** to deal with configuration changes when membership changed.

##### Section2:Replicated state meachines
- State machines on a collection of servers compute identical copies of the same state and can continue operating even if some of the servers are down.
-  Replicated state machines are used to solve a variety of fault tolerance problems in distributed systems.
-  Replicated state machine architecture
![](https://i.loli.net/2020/02/23/ZdcEgABWhQPv7yO.png)
- Procedure:
  - Replicated state machines are typically implemented using a replicated log
  -  Each server stores a log containing a series of commands, which its state machine executes in order.
- Consensus algorithms for practical systems typically have the following properties:
  - Consensus algorithms for practical systems typically have the following properties:
    - ***They are fully functional (available) as long as any majority of the servers are operational and can communicate with each other and with clients.*** Thus, a typical cluster of ﬁve servers can tolerate the failure of any two servers. Servers are assumed to fail by stopping; they may later recover from state on stable storage and rejoin the cluster.
    - ***They do not depend on timing to ensure the consistency of the logs:*** faulty clocks and extreme message delays can, at worst, cause availability problems.
    - ***In the common case, a command can complete as soon as a majority of the cluster has responded to a single round of remote procedure calls;*** a minority of slow servers need not impact overall system performance.
#### Section4:Designing for understandability
-  Divided problems into separate pieces
  - Separated the raft into ***Leader election, Log replication, Safety, and Membership changes.***
  - Reducing the number of states to consider, making the system more coherent and eliminating nondeterminism where possible.
    -  logs are not allowed to have holes, and Raft limits the ways in which logs can become inconsistent with each other.
    - We used randomization to simplify the Raft leader election algorithm.
#### Section5:The Raft consensus algorithm
- Raft implements consensus by ﬁrst electing a distinguished leader,  then giving the leader complete responsibility for managing the replicated log.
-  The leader accepts log entries from clients, replicates them on other servers, and tells servers when it is safe to apply log entries to their state machines.
- Raft decomposes the consensus problem into three relatively independent subproblems
  - ***Leader election***: a new leader must be chosen when an existing leader fails
  -  ***Log replication***: the leader must accept log entries from clients and replicate them across the cluster, forcing the other logs to agree with its own
  - ***Safety***: the key safety property for Raft is the State Machine Safety Property in Figure 3: if any server has applied a particular log entry to its state machine, then no other server may apply a different command for the same log index.
  ![](https://i.loli.net/2020/02/24/SINkRtpDr3mf1i2.png)

##### 5.1 Raft Basics
###### 5.1.1 Roles
- Each Server is in one of three states
  - leader
    -  The leader handles all client requests (if a client contacts a follower, the follower redirects it to the leader)
  - follower
    -  they issue no requests on their own but simply respond to requests from leaders and candidates
  - candidate
    -  The third state, candidate, is used to elect a new leader
###### 5.1.2 Terms
- Raft divides time into terms of arbitrary length, as shown in Figure 5. Terms are numbered with consecutive integers
  -  Each term begins with an election, in which one or more candidates attempt to become leader
  -  If a candidate wins the election, then it serves as leader for the rest of the term
  -  In some situations an election will result in a split vote
      - In that case: the term will end with no leader
      -  a new term (with a new election) will begin shortly

-  Raft ensures that there is at most one leader in a given term
-  Raft allow servers to detect obsolete information such as stale leaders.
- Each server stores a current term number, which increases monotonically over time.
- Current terms are exchanged whenever servers communicate;
- if one server’s current term is smaller than the other’s, then it updates its current term to the larger value.
- If a candidate or leader discovers that its term is out of date, it immediately reverts to follower state.
- If a server receives a request with a stale term number, it rejects the request.
###### 5.1.3 RPC
-  The basic consensus algorithm requires only two types of RPCs
  - ***RequestVote RPCs*** are initiated by candidates during elections
  -  ***AppendEntries RPCs*** are initiated by leaders to replicate log entries and to provide a form of heartbeat
  - The third RPC for transferring snapshots between servers
##### 5.2 Leader election
- Raft uses a heartbeat mechanism to trigger leader election.
-  When servers start up, they begin as followers. A server remains in follower state as long as it receives valid RPCs from a leader or candidate
-  Leaders send periodic heartbeats (AppendEntriesRPCs that carry no log entries) to all followers in order to maintain their authority
-  If a follower receives no communication over a period of time called the election timeout, then it assumes there is no viable leader and begins an election to choose a new leader.
###### begin elections
- To begin an election, a follower increments its current term and transitions to candidate state.
- It then votes for itself and issues RequestVote RPCs in parallel to each of the other servers in the cluster
- A candidate continues in this state until one of three things happens:
  - it wins the election
  - another server establishes itself as leader
  - a period of time goes by with no winner
###### after beginning
- A candidate wins an election if it receives votes from a majority of the servers in the full cluster for the same term
-  Each server will vote for at most one candidate in a given term, on a ﬁrst-come-ﬁrst-served basis
- The majority rule ensures that at most one candidate can win the election for a particular term
-  Once a candidate wins an election, it becomes leader. It then sends heartbeat messages to all of the other servers to establish its authority and prevent new elections.
###### while waiting
- While waiting for votes, a candidate may receive an AppendEntries RPC from another server claiming to be leader.
  -  If the leader’s term (included in its RPC) is at least as large as the candidate’s currentterm, then the candidate recognizes the leader as legitimate and returns to follower state
  -  If the term in the RPC is smaller than the candidate’s current term, then the candidate rejects the RPC and continues in candidate state.
- if many followers become candidates at the same time, votes could be split so that no candidate obtains a majority
  -  When this happens, each candidate will time out and start a new election by incrementing its term and initiating another round of RequestVote RPCs. However, without extra measures split votes could repeat indeﬁnitely.
  - Raft uses ***randomized election timeouts*** to ensure that split votes are rare and that they are resolved quickly
    -  To prevent split votes in the ﬁrst place, election timeouts are chosen randomly from a ﬁxed interval (e.g., 150–300ms).
    - This spreads out the servers so that in most cases only a single server will time out
    -  it wins the election and sends heartbeats before any other servers time out
  - The same mechanism is used to handle split votes
    -  Each candidate restarts its randomized election timeout at the start of an election, and it waits for that timeout to elapse before starting the next election
    -  this reduces the likelihood of another split vote in the new election
##### 5.3 Log replication
