Caching & Rate Limiting — System Design Interview Notes

⸻

1. Why Cache?

Problem

Without cache:

User
 ↓
Database
 ↓
50 ms response

At 1000 requests/sec:

1000 × 50ms = DB bottleneck

Database becomes overloaded.

⸻

With Cache

User
 ↓
Redis
 ↓
1 ms response

Benefits:

✅ 50x faster

✅ Reduces DB load

✅ Better scalability

✅ Better user experience

Tradeoffs:

❌ Stale data

❌ Memory cost

❌ Invalidation complexity

⸻

2. Cache Hierarchy

CPU L1 Cache
    ↓
CPU L2/L3
    ↓
RAM
    ↓
Redis/Memcached
    ↓
Database
    ↓
Disk

Access speed:

Storage	Latency
L1 Cache	1 ns
RAM	100 ns
Redis	~1 ms
SSD	0.1 ms
HDD	10 ms
Network	1-100 ms

⸻

3. Cache Patterns

⸻

A. Cache-Aside (Lazy Loading)

Flow

Request
   ↓
Check Cache
   ↓
Hit?
 ├── YES → Return
 └── NO
        ↓
      Query DB
        ↓
      Store Cache
        ↓
       Return

Advantages

✔ Simple

✔ DB fallback exists

✔ Cache only active data

Disadvantages

✘ First request slow

✘ Stale data

✘ Stampede risk

Best For

Read-heavy systems

Examples:

* Product pages
* User profiles
* News feed

⸻

B. Write Through

Flow

Application
      ↓
Cache
      ↓
Database

Advantages

✔ Cache always consistent

✔ Reads are fast

Disadvantages

✘ Write latency = DB latency

✘ Extra cache operations

Best For

Critical data

Examples:

* Banking
* Inventory

⸻

C. Write Behind

Flow

Application
     ↓
Cache
     ↓
Return immediately
Background Worker
     ↓
Database

Advantages

✔ Very fast writes

✔ Batch updates

✔ Less DB pressure

Disadvantages

✘ Possible data loss

✘ Eventual consistency

Best For

Logs

Analytics

Metrics

⸻

D. Refresh Ahead

Before expiry:

Cache Entry
      ↓
TTL nearing end
      ↓
Background Refresh
      ↓
Fresh Cache

Advantages

✔ No cache misses

✔ Fresh data

Disadvantages

✘ Extra DB queries

Best For

Hot data

Example:

Netflix popular videos

⸻

Decision Tree

Mostly Reads?
     ↓
YES
 ↓
Cache Aside
Need Strong Consistency?
     ↓
YES
 ↓
Write Through
Need Very Fast Writes?
     ↓
YES
 ↓
Write Behind
Hot Data?
     ↓
YES
 ↓
Refresh Ahead

⸻

4. Eviction Policies

When cache becomes full:

Cache Size = 1GB
New Item arrives
Need eviction

⸻

LRU (Least Recently Used)

Access:
A B C A B
Recent Order:
B A C
Evict C

Best General Purpose Policy

Examples:

* Redis
* Browser cache

⸻

LFU (Least Frequently Used)

A = 100 accesses
B = 50 accesses
C = 5 accesses
Evict C

Best For

Hot objects

Example:

YouTube trending videos

⸻

FIFO

Inserted:
A B C
Evict A

Simple but inefficient.

⸻

Random

Random eviction.

Very cheap.

⸻

Eviction Policy Decision

General Use
     ↓
LRU
Hot Objects
     ↓
LFU
Simplicity
     ↓
FIFO
High Throughput
     ↓
Random

⸻

5. Distributed Cache

Single Redis:

Redis
16 GB RAM
100k ops/sec

Problem:

❌ Single point of failure

❌ Memory limited

⸻

Redis Cluster

       Client
          ↓
    hash(key)%N
┌─────────────┐
│ Node 1      │
├─────────────┤
│ Node 2      │
├─────────────┤
│ Node 3      │
└─────────────┘

Benefits:

10 nodes
Memory:
10 × 16GB = 160GB
Throughput:
10 × 100k = 1M ops/sec

Replication:

Master
   ↓
Replica

Automatic failover.

⸻

6. Cache Invalidation

There are only two hard problems:

Cache invalidation and naming things.

⸻

TTL

cache.set(key, value, ttl=1hr)

Simple.

But data may remain stale for one hour.

⸻

Explicit Invalidation

DB update
 ↓
Delete cache

Danger:

Race condition.

⸻

Event-Based

Database Update
        ↓
Kafka
        ↓
Consumer
        ↓
Invalidate Cache

Reliable and scalable.

⸻

Production Recommendation

Data Type	Strategy
Critical	Event + Write Through
Normal	TTL + Event
Non-Critical	TTL

⸻

7. Cache Stampede

Problem

Popular key expires.

10000 requests
      ↓
Cache miss
      ↓
10000 DB queries
      ↓
DB crash

⸻

Solutions

⸻

Locking

Request 1
    ↓
Acquire Lock
    ↓
DB Query
Request 2
    ↓
Wait
Request 3
    ↓
Wait

Only one query reaches DB.

⸻

Probabilistic Refresh

Refresh before expiration.

Randomized refresh avoids synchronized expiry.

⸻

XFetch

Two TTLs:

Soft TTL

Return stale value and refresh in background.

Hard TTL

Force DB query.

0---------50min-------60min
Fresh       Soft       Hard
             Expire     Expire

Used in large-scale systems.

⸻

8. Rate Limiting

Purpose:

Prevent abuse.

Guarantee fairness.

Protect servers.

⸻

Token Bucket ⭐ (Most Important)

Capacity = 100 tokens
Each request consumes 1 token

No token:

Reject request

⸻

Example

Refill rate = 100/sec
Capacity = 1000
Can burst to 1000 requests
Then sustain:
100 req/sec

Best For

APIs

Most common interview answer.

⸻

Sliding Window Log

Stores every timestamp.

Request times:
0.1
0.2
0.3
...

Count requests inside window.

Advantages

Exact

Disadvantages

Memory expensive

⸻

Sliding Window Counter

Maintains:

Previous Window Count
Current Window Count

Interpolates between them.

Advantages

Efficient

Disadvantages

Approximate

⸻

Rate Limiting Decision Tree

Need burst support?
      ↓
YES
 ↓
Token Bucket ⭐
Need exact limits?
      ↓
YES
 ↓
Sliding Window Log
Need low memory?
      ↓
YES
 ↓
Sliding Window Counter

⸻

9. Distributed Rate Limiting

Problem:

10 servers

100 req/server

Actual:

1000 req/sec

Limit violated.

⸻

Redis Counter

Server
   ↓
INCR user123
   ↓
Redis

Global count maintained.

Pros

✔ Accurate

Cons

✘ Redis bottleneck

⸻

Local + Sync

Server1 count = 10
Server2 count = 12
Sync every 100ms

Pros

Fast

Cons

Temporary overshoot possible

⸻

Gossip Protocol

Server1 → Server2
Server2 → Server3
Server3 → Server4

Eventually converges.

Used in distributed systems.

⸻

10. What FAANG Interviewers Expect

If asked:

“How would you cache product pages?”

Answer:

Cache Aside
+
Redis
+
TTL
+
LRU
+
Stampede protection

⸻

“How would you cache bank balances?”

Answer:

Write Through
+
Event-based invalidation

⸻

“How would you rate limit APIs?”

Answer:

Token Bucket
+
Redis counter

⸻

“How do you avoid cache stampedes?”

Answer:

Mutex lock
OR
Soft TTL + Background refresh
OR
XFetch

⸻

Production Stack Example

Client
   ↓
Load Balancer
   ↓
Application Servers
   ↓
Redis Cluster
   ↓
MySQL/Postgres

Cache

* Cache Aside
* TTL = 1 hr
* LRU eviction

Stampede Prevention

* Mutex Lock
* Soft TTL

Rate Limiting

* Token Bucket
* Redis Counter

Monitoring

Target:

Cache Hit Ratio > 80%
P95 latency < 100ms
Redis memory utilization < 75%

⸻

Golden Rules

1

Cache is a performance optimization, not source of truth.

⸻

2

Cache invalidation is harder than caching.

⸻

3

High hit ratio (>80%) is crucial.

⸻

4

Protect databases from stampedes.

⸻

5

Token Bucket is the default rate limiter.

⸻

6

LRU is the default eviction policy.

⸻

7

Redis Cluster scales horizontally.

⸻

8

Always plan cache invalidation before adding cache.

⸻

