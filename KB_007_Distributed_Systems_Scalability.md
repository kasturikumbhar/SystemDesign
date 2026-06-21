📘 Knowledge Base: Distributed Systems & Scalability

Replication • Consensus • CAP Theorem • Sharding • Eventual Consistency • Production Patterns

⸻

🏗 PART 1 — DISTRIBUTED SYSTEM FUNDAMENTALS

⸻

🟦 Why Distributed Systems?

Single machine eventually hits physical limits

                Single Machine
      ┌────────────────────────────┐
      │ CPU      ~10k req/sec/core │
      │ Memory   ~256GB            │
      │ Disk IOPS ~100k SSD        │
      │ Availability = SPOF        │
      └────────────────────────────┘
                     │
                     ▼
            Need Horizontal Scaling
                     │
                     ▼
        ┌────────────┬────────────┬────────────┐
        │ Machine A  │ Machine B  │ Machine C  │
        └────────────┴────────────┴────────────┘

🟩 Benefits

Problem	Solution
CPU bottleneck	More machines
Memory limits	Distributed storage
Disk limits	Sharding
Single point of failure	Replication
Scale to millions	Cluster

⸻

🟥 New Problems Introduced

Distributed Systems
        Machines
           │
 ┌─────────┼─────────┐
 │         │         │
 ▼         ▼         ▼
Latency  Failures  Consistency

Challenges

🟨 Network latency

🟨 Partial failures

🟨 Coordination

🟨 Clock differences

🟨 Multiple copies of data

⸻

⚠️ Failure Modes

Machine Failure

Node A
   │
   ▼
CRASH
   │
Restart
   │
Rejoin cluster

⸻

Network Partition

Cluster
Node A ─────X───── Node B
             |
          connection lost
Both alive
Cannot communicate

⸻

Cascading Failure

Service A slow
      │
      ▼
Retries increase
      │
      ▼
Service B overloaded
      │
      ▼
More retries
      │
      ▼
Entire system down

⸻

🔵 PART 2 — REPLICATION

⸻

Leader-Follower Replication

                    Writes
Client ───────────────► Leader
                           │
                Replicate updates
                 ┌─────────┴─────────┐
                 ▼                   ▼
             Follower A          Follower B
Reads can go anywhere
Writes only to Leader

⸻

🟩 Advantages

✅ Simple

✅ Read scaling

✅ High availability

⸻

🟥 Drawbacks

❌ Leader bottleneck

❌ Replication lag

❌ Split-brain risk

⸻

Multi-Master Replication

            Node A
           ↔     ↔
         ↔         ↔
      Node B ↔ Node C
Every node accepts reads + writes

Benefits

🟩 No single leader

🟩 Higher write availability

⸻

Problem

Concurrent updates:

Node A:
Age = 30
Node B:
Age = 31
Conflict!

Requires conflict resolution.

⸻

PART 3 — REPLICATION LAG

⸻

Write Flow

Client
   │
   ▼
Leader receives write
   │
0ms
   │
▼
Follower receives
10ms
   │
▼
Apply update
12ms

⸻

During These 12ms

Leader:
Name = John
Follower:
Name = ""
(stale)

⸻

Problem 1

Read-after-write inconsistency

User updates profile
Write → Leader
Immediately read
Read → Follower
Old value returned

Solution

🟩 Route recent reads to leader

⸻

Problem 2

Monotonic Read Violation

Read 1:
Version 5
Read 2:
Version 3
Time appears to move backward

Solution

🟩 Sticky sessions

⸻

🔴 PART 4 — CAP THEOREM

⸻

Three Properties

          CAP
         / | \
        /  |  \
       C   A   P

⸻

🟦 Consistency

Every node sees latest value.

Read Node A = 5
Read Node B = 5
Read Node C = 5

⸻

🟩 Availability

System always responds.

Request
   │
Response always returned

⸻

🟨 Partition Tolerance

Network failures tolerated.

Node A  X  Node B
System continues

⸻

CAP Reality

Network partitions WILL happen

Therefore:

P is mandatory
Choose:
CP or AP

⸻

CP Systems

Consistency
+
Partition Tolerance

Sacrifice availability.

Examples:

🟦 PostgreSQL

🟦 ZooKeeper

🟦 etcd

⸻

During Leader Failure

Leader down
No writes allowed
Wait for recovery

⸻

AP Systems

Availability
+
Partition Tolerance

Sacrifice consistency.

Examples:

🟩 Cassandra

🟩 DynamoDB

⸻

During partition:

Both sides continue accepting writes
Temporary inconsistency

⸻

🟣 PART 5 — CONSENSUS (RAFT)

⸻

Goal

All nodes agree on same sequence.

⸻

Cluster

          Leader
            │
     ┌──────┼──────┐
     ▼      ▼      ▼
Follower Follower Follower

⸻

Leader Election

Leader dies
     │
Followers wait random timeout
     │
One becomes Candidate
     │
Requests votes
     │
Majority wins
     │
New Leader elected

⸻

Log Replication

Client
  │
  ▼
Leader appends entry
  │
  ▼
Send to followers
  │
  ▼
Majority ACK
  │
  ▼
Commit

⸻

Example

5-node cluster

Need majority = 3
Node1 Leader
Node2 Follower
Node3 Follower
Node4 Follower
Node5 Down
3 acknowledgements
COMMIT SUCCESS

Can tolerate:

2 failures

⸻

🟠 PART 6 — SHARDING

⸻

Problem

Year1 → 100GB
Year2 → 1TB
Year3 → 10TB
Single machine no longer sufficient

⸻

Range Sharding

Shard1
1 → 1M
Shard2
1M → 2M
Shard3
2M → 3M

Pros

🟩 Easy range queries

Cons

🟥 Hot ranges possible

⸻

Hash Sharding

hash(user_id)%3
500k → Shard1
1.5M → Shard2
2.5M → Shard3

Pros

🟩 Uniform distribution

Cons

🟥 Difficult range queries

⸻

Consistent Hashing

           A
        /     \
     D           B
        \      /
           C

Keys placed around ring.

Adding new node:

Only nearby keys move

Very little rebalancing.

Used by:

🟢 DynamoDB

🟢 Cassandra

⸻

🔥 Hot Shard Problem

Shard1
Celebrity user
1M reads/sec
Shard2
10k reads/sec
Shard3
10k reads/sec

Result:

Shard1 overloaded
Others idle

⸻

Solutions

🟩 Cache

Users
   │
Redis Cache
   │
Database

⸻

🟩 Read Replicas

          Primary
        /     |     \
Replica1 Replica2 Replica3

⸻

🟩 Sub-sharding

Shard1
→ Shard1A
→ Shard1B
→ Shard1C

⸻

🟡 PART 7 — EVENTUAL CONSISTENCY

⸻

Timeline

T = 0

Node A = 5
Node B = 0
Node C = 0

⸻

T = 100ms

Node A = 5
Node B = 5
Node C = 0

⸻

T = 200ms

Node A = 5
Node B = 5
Node C = 5

System converges.

⸻

Definition

If no new writes occur, eventually all replicas become identical.

⸻

Suitable For

🟩 Social media feeds

🟩 Analytics

🟩 Caching

⸻

Not Suitable For

🟥 Banking

🟥 Inventory

🟥 Authentication

⸻

⚔ Conflict Resolution

⸻

Last Write Wins

A : age=30 @100
B : age=31 @99
Keep timestamp 100

Simple but clock dependent.

⸻

Vector Clocks

Node A
[A:5 B:3]
Node B
[A:3 B:4]

Tracks causality.

More accurate.

⸻

CRDT

Concurrent updates naturally merge.

Example:

Counter
Node A +5
Node B +3
Result = +8

No conflicts.

⸻

🔵 PART 8 — OBSERVABILITY

⸻

Three Pillars

        Observability
     ┌────────┼─────────┐
     ▼        ▼         ▼
 Metrics    Logs     Traces

⸻

Metrics

WHAT happened?

Request count
Error %
P99 latency

Tools:

🟢 Prometheus

🟢 CloudWatch

⸻

Logs

WHY happened?

ERROR:
Payment timeout
Order ID=1234

Tools:

🟢 ELK

🟢 Splunk

🟢 Loki

⸻

Traces

HOW did it happen?

Gateway
   │
Order Service
   │
Payment Service (timeout)
   │
Notification

Tools:

🟢 Jaeger

🟢 Zipkin

⸻

🚨 PART 9 — ALERTING

⸻

Critical (Page Immediately)

🔴 Service down

🔴 Data loss risk

🔴 500 errors

Pager → On-call engineer

⸻

Warning (Ticket)

🟡 High latency

🟡 Replication lag

Investigate during work hours.

⸻

Informational

🟢 Deployment completed

🟢 Scaling event

No alert needed.

⸻

📌 CHEAT SHEET

Concept	Purpose
Replication	High availability
Consensus (Raft)	Agreement among nodes
CAP	C vs A under partition
Sharding	Horizontal scaling
Consistent Hashing	Easy rebalancing
Eventual Consistency	High availability
CRDTs	Conflict-free updates
Metrics	WHAT
Logs	WHY
Traces	HOW
Alerts	Detect failures

⸻

⭐ GOLDEN RULES

Design for failure.

Assume networks are unreliable.

Replication gives availability, not consistency.

CAP means partitions force tradeoffs.

Sharding solves scale but adds complexity.

Monitor everything.

Start simple; distribute only when necessary.

⸻

🧠 Mental Model

                Distributed System
      Scale
         │
         ▼
     Sharding
         │
         ▼
   Multiple Machines
         │
         ▼
     Replication
         │
         ▼
     Consistency
         │
         ▼
      Consensus
         │
         ▼
      Observability
         │
         ▼
      Reliability

⸻

The hardest part of distributed systems isn’t making them work — it’s making them fail gracefully.
