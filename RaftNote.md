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
- Once a leader has been elected, it begins servicing client requests. Each client requestcontainsa commandto be executed by the replicated state machines
-  The leader appends the command to its log as ***a new entry***
-  then issues AppendEntries RPCs in parallel to each of the other servers to replicate the entry
-  When the entry has been safely replicated (as described below), the leader applies the entry to its state machine and returns the result of that execution to the client
- If followers crash or run slowly, or if network packets are lost, the leader retries AppendEntries RPCs indeﬁnitely (even after it has responded to the client) until all followers eventually store all log entries.
- Logs are organized as shown in Figure 6.
- The term numbers in log entries are used to detect inconsistencies between logs and to ensure some of the properties in Figure 3.
- Each log entry also has an integer index identifying its position in the log.
![](https://i.loli.net/2020/02/24/eWgwolKcFb18TLy.png)

###### 5.3.2 when to commit
- The leader decides when it is safe to apply a log entry to the state machines，such an entry is called ***committed.***
-  Raft guarantees that committed entries are durable and will eventually be executed by all of the available state machines.
-  A log entry is committed once the leader that created the entry has replicated it on a majority of the servers
- This also commits all preceding entries in the leader’s log, including entries created by previous leaders
-  The leader keeps track of the highest index it knows to be committed, and it includes that index in future AppendEntries RPCs (including heartbeats) so that the other servers eventually ﬁnd out
-  Once a follower learns that a log entry is committed, it applies the entry to its local state machine (in log order).
- Raft maintains the following properties, which together constitute the Log Matching Property in Figure 3:
  -  If two entries in different logs have the same index and term, then they store the same command
      - The property follows from the fact that a leader creates at most one entry with a given log index in a given term, and log entries ***never change their position*** in the log.
  -  If two entries in different logs have the same index and term, then the logs are identical in all preceding entries.
      -  The second property is guaranteed by a simple consistency check performed by AppendEntries. When sending an AppendEntries RPC, the leader ***includes the index and term of the entry*** in its log
      - (?) If the follower does not ﬁnd an entry in its log with the same index and term, then it refuses the new entries.
###### 5.3.3 Leader fail
-  leader crashes can leave the logs inconsistent (the old leader may not have fully replicated all of the entries in its log).
    - In Raft, the leader handles inconsistencies by forcing the followers’ logs to duplicate its own
    - This means that conﬂicting entries in follower logs will be overwritten with entries from the leader’s log
    - To bring a follower’s log into consistency with its own, the leader must ﬁnd the latest log entry **where the two logs agree**, delete any entries in the follower’s log **after that point**, and send the follower all of the leader’s entries after that point
      - **Question: If this mean that the entries after that will be abandoned?**
    -  All of these actions happen in response to the consistency check performed by AppendEntries RPCs.
    -  The leader maintains a **nextIndex** for each follower, which is the index of the next log entry the leader will send to that follower
    -  When a leader ﬁrst comes to power, it initializes all nextIndex values to the index just after the last one in its log (11 in Figure 7).
        - If a follower’s log is inconsistent with the leader’s, the AppendEntries consistency check will fail in the next AppendEntries RPC
        - After a rejection,the leader decrementsnextIndexand retries the AppendEntries RPC
        - Eventually nextIndex will reach a point where the leader and follower logs match
        -  When this happens,AppendEntrieswill succeed, which removes any conﬂicting entries in the follower’s log and appends entries from the leader’s log (if any).
        -  Once AppendEntries succeeds, the follower’slog is consistentwith the leader’s, and it will remain that way for the rest of the term.
##### 5.4 safety
- A follower might be unavailable while the leader commits several log entries, then it could be elected leader and overwrite these entries with new ones;
- as a result, different state machines might execute different command sequences.
###### 5.4.1 election Restriction
-  Raft uses a simpler approach where it guarantees that all the committed entries from previous terms are present on each new leader from the moment of its election
-  This means that log entries only ﬂow in one direction, from leaders to followers, and leaders never overwrite existing entries in their logs.
- Raft uses the voting processto preventa candidate from winning an election unless its log contains all committed entries
  - A candidate must contact a majority of the cluster in order to be elected, which means that every committed entry must be present in ***at least one of those servers***
  -  If the candidate’s log is at least as up-to-date as any other log in that majority then it will hold all the committed entries
  - The RequestVote RPC implements this restriction:**the RPC includes information about the candidate’s log, and the voter denies its vote if its own log is more up-to-date than that of the candidate.**
- Raft determines which of two logs is more up-to-date by comparing the index and term of the last entries in the logs.
  - If the logs have last entries with different terms, then the log with the later term is more up-to-date.
  - If the logs end with the same term, then whichever log is longer is more up-to-date.
###### Committing entries from previous terms
-  A new leader cannot immediately conclude that an entry from a previous term is committed once it is stored on a majority of servers, and it can still be overwritten by a future leader.
- To eliminate problems like the one in Figure 8, Raft never commits log entries from previous terms by counting replicas. Only log entries from the leader’s current term are committed by counting replicas
##### 5.5 Follower and candidate crashes
- If a follower or candidate crashes, then future RequestVote and AppendEntries RPCs sent to it will fail. Raft handles these failures by retrying indeﬁnitely;
- if the crashed server restarts, then the RPC will complete successfully.
- If a server crashes after completing an RPC but before responding, then it will receive the same RPC again after it restarts. Raft RPCs are idempotent, so this causes no harm.
##### 5.6  Timing and availability
-  Raft is that safety must not depend on timing
- withouta steady leader, Raft cannotmake progress.
- Leader election is the aspect of Raft where timing is most critical.
-  Raft will be able to elect and maintain a steady leader as long as the system satisﬁes the following timing requirement:
    - broadcastTime << electionTimeout << MTBF
    - In this inequality **broadcastTime** is the average time it takes a server to send RPCs in parallel to every server in the cluster and receive their responses
    - electionTimeout is the election timeout
    -  and MTBF is the average time between failures for a single server.
    - The **broadcast time** should be an order of **magnitude less** than the election timeout so that leaders can reliably send the heartbeat messages required to **keep followers from starting elections**
    -  given the randomized approach used for election timeouts, this inequality also makes split votes unlikely
    - The **election timeout** should be a few orders of **magnitude less** than MTBF so that the system makes **steady progress**
-  the broadcast time may range from 0.5ms to 20ms, depending on storage technology
- the election timeout is likely to be somewherebetween 10ms and 500ms
- MTBFs are several months or more, which easily satisﬁes the timing requirement.
#### Cluster membership changes
- use a two-phase approach
