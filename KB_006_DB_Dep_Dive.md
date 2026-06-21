⸻

📘 KB_006: Databases Deep Dive

⸻

DATABASE ARCHITECTURE

Application
      │
Connection Pool
      │
Query Optimizer
      │
Execution Engine
      │
Storage Engine
      │
Disk

⸻

PART 1 — STORAGE ENGINES

⸻

B-Tree

(MySQL)

          Root
       /    |    \
      ▼     ▼     ▼
 Internal Nodes
      ▼     ▼     ▼
 Leaf Nodes

Complexity:

Search = O(log n)

Excellent for:

✅ Range queries

✅ ORDER BY

⸻

LSM Tree

(Cassandra, RocksDB)

Write
 │
 ▼
MemTable
↓
SSTable
↓
Compaction

Great for:

✅ Heavy writes

❌ Expensive reads

⸻

B-Tree vs LSM

Feature	BTree	LSM
Reads	Fast	Medium
Writes	Medium	Fast
Range scan	Excellent	Weak
Compaction	No	Yes
MySQL	✅	❌
Cassandra	❌	✅

⸻

PART 2 — INDEXES

⸻

Primary Index

ID → Row

Clustered.

⸻

Secondary Index

Email → ID

Needs extra lookup.

⸻

Composite Index

(age, city)

Works:

WHERE age=20
WHERE age=20 AND city='NY'

Not:

WHERE city='NY'

(leftmost rule)

⸻

Covering Index

Everything inside index.

Index contains:
Name
Age
Salary

No table access required.

⸻

Partial Index

WHERE status='ACTIVE'

Smaller and faster.

⸻

PART 3 — QUERY EXECUTION

SQL Query
↓
Parser
↓
Optimizer
↓
Execution Plan
↓
Storage Engine

⸻

EXPLAIN

Bad:

FULL TABLE SCAN
10M rows

Good:

INDEX SEEK
100 rows

⸻

PART 4 — ACID

⸻

Atomicity

All or Nothing

⸻

Consistency

Constraints preserved.

⸻

Isolation

Transactions independent.

⸻

Durability

Data survives crash.

⸻

PART 5 — ISOLATION LEVELS

⸻

Read Uncommitted

Dirty reads possible.

⸻

Read Committed

Most databases.

No dirty reads.

⸻

Repeatable Read

Same query returns same rows.

⸻

Serializable

Strongest.

Slowest.

⸻

PART 6 — MVCC

Row v1
↓
Update
↓
Row v2
↓
Old version retained

Readers never block writers.

⸻

Snapshot

T1 sees Version 1
T2 updates Version 2
T1 still sees Version 1

⸻

PART 7 — N+1 PROBLEM

⸻

Bad:

1 query users
100 queries orders
101 queries total

⸻

Solution:

JOIN FETCH

SELECT *
FROM users
JOIN orders

Only one query.

⸻

PART 8 — CONNECTION POOL

Threads
↓
HikariCP
↓
Reusable Connections
↓
Database

Avoids expensive connection creation.

⸻

PART 9 — QUERY TUNING

⸻

Step 1

EXPLAIN query

↓

Full table scan?

↓

Add index

↓

Recheck plan

↓

Measure latency

⸻

Step 2

Avoid:

SELECT *

Return only required columns.

⸻

Step 3

Pagination

Bad:

OFFSET 100000

Good:

WHERE id > last_id
LIMIT 20

(Keyset Pagination)

⸻

Step 4

Cache hot queries

App
↓
Redis
↓
Database

⸻

PART 10 — DEADLOCKS

Transaction A
locks row1
needs row2
Transaction B
locks row2
needs row1
DEADLOCK

Database aborts one transaction.

⸻

PART 11 — REPLICATION

Primary
      │
 ┌────┴────┐
 ▼         ▼
Replica1 Replica2

Writes:

Primary

Reads:

Replicas

⸻

PART 12 — SHARDING

Users 1-1M
Shard1
Users 1M-2M
Shard2
Users 2M-3M
Shard3

Horizontal scaling.

⸻

🧠 Database Mental Model

Query
 │
 ▼
Optimizer
 │
 ▼
Index
 │
 ▼
Storage Engine
 │
 ▼
Disk
 │
 ▼
Replication
 │
 ▼
Scaling

⸻

⭐ Ultimate Principle

JVM performance problems are mostly memory problems.

Database performance problems are mostly indexing problems.
