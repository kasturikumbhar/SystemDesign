
I’ll format it like a Notion page, with callout boxes, diagrams, tables, and interview notes. This style is designed for quick revision before system design interviews.

⸻

🧠 Caching & Rate Limiting

Goal: Faster responses, lower DB load, protect services from overload.

⸻

📑 Table of Contents

1. Why Caching?
2. Cache Patterns
3. Eviction Policies
4. Distributed Cache
5. Cache Invalidation
6. Cache Stampede
7. Rate Limiting
8. Distributed Rate Limiting
9. Production Architecture
10. Interview Cheat Sheet

⸻

⚡ 1. Why Cache?

Without Cache

User
 ↓
Application
 ↓
Database (50ms)
 ↓
Response

🔴 Every request hits DB

🔴 Database becomes bottleneck

⸻

With Cache

User
 ↓
Application
 ↓
Redis (1ms)
 ↓
Response

🟢 Faster responses

🟢 Reduced DB load

🟢 Higher throughput

⸻

💡 Rule

Cache is a performance optimization, NOT the source of truth.

⸻

📦 2. Cache Patterns

⸻

🟦 Cache Aside (Lazy Loading)

Flow

Request
  ↓
Cache?
 ├── Hit → Return
 │
 └── Miss
        ↓
      Database
        ↓
      Cache Data
        ↓
       Return

✅ Advantages

🟢 Simple

🟢 Only caches active data

🟢 DB fallback available

❌ Disadvantages

🔴 First request slow

🔴 Stale data

🔴 Cache stampede risk

⸻

Best Use Cases

📌 Product pages

📌 User profiles

📌 News feed

📌 Read-heavy systems

⸻

🟩 Write Through

Flow

Application
     ↓
 Cache
     ↓
 Database

Pros

🟢 Strong consistency

🟢 Fast reads

Cons

🔴 Write latency = DB latency

⸻

Best For

🏦 Banking

📦 Inventory systems

⸻

🟨 Write Behind

Flow

Application
      ↓
Cache
      ↓
Return immediately
Background Worker
      ↓
Database

Pros

🟢 Extremely fast writes

🟢 Batch updates

🟢 Less DB pressure

Cons

🔴 Possible data loss

🔴 Eventual consistency

⸻

Best For

📊 Analytics

📜 Logs

📈 Metrics

⸻

🟪 Refresh Ahead

Before TTL expires:

Cache Entry
      ↓
Near Expiration
      ↓
Background Refresh
      ↓
Fresh Cache

Best For

🔥 Hot Data

🎥 Popular Videos

⸻

🎯 Pattern Selection

Mostly Reads?
      ↓
Cache Aside
Need Strong Consistency?
      ↓
Write Through
Need Fast Writes?
      ↓
Write Behind
Hot Data?
      ↓
Refresh Ahead

⸻

🗑️ 3. Cache Eviction Policies

When cache becomes full:

Cache Full
    ↓
Need Eviction

⸻

🟦 LRU (Least Recently Used)

Access:
A B C A B
Order:
B A C
Evict C

✅ Most common

✅ General-purpose

Used By

Redis

Browser cache

⸻

🟨 LFU (Least Frequently Used)

A = 100 accesses
B = 50 accesses
C = 5 accesses
Evict C

Best For

🔥 Popular objects

YouTube videos

Trending feeds

⸻

🟩 FIFO

Inserted:
A → B → C
Evict A

Simple but inefficient.

⸻

🎯 Eviction Decision Tree

General Purpose
     ↓
LRU ⭐
Hot Objects
     ↓
LFU
Simple System
     ↓
FIFO

⸻

🌐 4. Distributed Cache

Single Redis

16 GB RAM
100k ops/sec

Problems:

🔴 Single point of failure

🔴 Limited memory

⸻

Redis Cluster

           Client
              ↓
        hash(key)%N
 ┌──────────────┐
 │ Redis Node 1 │
 ├──────────────┤
 │ Redis Node 2 │
 ├──────────────┤
 │ Redis Node 3 │
 └──────────────┘

Benefits

🟢 Horizontal scaling

🟢 High throughput

🟢 Fault tolerance

⸻

Replication

Master
  ↓
Replica

Automatic failover.

⸻

⭐ Redis Cluster = Standard interview answer

⸻

🔄 5. Cache Invalidation

💀 Hardest problem in Computer Science.

⸻

TTL

cache.set(key, value, ttl=1hr)

Pros

🟢 Simple

Cons

🔴 Stale data

⸻

Explicit Invalidation

DB Update
    ↓
Delete Cache

Danger

Race conditions

⸻

Event-Based Invalidation

Database
    ↓
Kafka
    ↓
Consumer
    ↓
Invalidate Cache

Best Practice

Data Type	Strategy
Critical	Event + Write Through
Normal	TTL + Events
Non-critical	TTL

⸻

🚨 6. Cache Stampede

Problem

Popular key expires.

10,000 users arrive.

10000 Requests
      ↓
Cache Miss
      ↓
10000 DB Queries
      ↓
DB Crash

⸻

Solution 1: Lock

Request 1
 ↓
Lock
 ↓
DB
Request 2
 ↓
Wait
Request 3
 ↓
Wait

Only one query reaches DB.

⸻

Solution 2: Early Refresh

Refresh before expiration.

Prevents synchronized misses.

⸻

Solution 3: XFetch

0--------------50m----------60m
Fresh         Soft TTL      Hard TTL
              Return stale
              + refresh

Production Favorite ⭐

⸻

🚦 7. Rate Limiting

Purpose:

🛡 Prevent abuse

⚖ Fair resource usage

🚀 Stable performance

⸻

🟦 Token Bucket ⭐

Most popular algorithm.

Capacity = 100
Request
 ↓
Consume token
No token?
 ↓
Reject

⸻

Burst Support

Capacity = 1000
Refill = 100/sec

Can burst to 1000 requests.

Then sustain 100/sec.

⸻

Best For

API gateways

Microservices

⸻

🟨 Sliding Window Log

Stores every timestamp.

0.1
0.2
0.3
0.4
...

Pros

🟢 Exact

Cons

🔴 High memory

⸻

🟩 Sliding Window Counter

Stores:

Previous Window
Current Window

Interpolates between them.

Pros

🟢 Efficient

Cons

🔴 Approximate

⸻

🎯 Rate Limiting Selection

Need burst support?
      ↓
Token Bucket ⭐
Need exact limits?
      ↓
Sliding Window Log
Need low memory?
      ↓
Sliding Window Counter

⸻

🌎 8. Distributed Rate Limiting

Problem:

10 servers

100 req/server

Actual:

1000 requests/sec

Limit broken.

⸻

Redis Counter

Server
   ↓
INCR user123
   ↓
Redis

Pros

🟢 Accurate

Cons

🔴 Redis bottleneck

⸻

Local + Sync

Server1 = 10
Server2 = 12
Sync every 100ms

Fast but approximate.

⸻

Gossip Protocol

Server1
   ↓
Server2
   ↓
Server3
   ↓
Server4

Eventually converges.

⸻

🏗 Production Architecture

                Users
                  │
                  ▼
          Load Balancer
                  │
                  ▼
          Application Servers
                  │
          ┌───────┴────────┐
          ▼                ▼
      Redis Cluster     Rate Limiter
          │                │
          └───────┬────────┘
                  ▼
            MySQL/Postgres
                  │
                  ▼
              Kafka Events
                  │
                  ▼
          Cache Invalidation

⸻

⭐ Interview Cheat Sheet

Cache Product Pages

Cache Aside
+
Redis
+
TTL
+
LRU
+
Stampede Protection

⸻

Bank Balances

Write Through
+
Event-Based Invalidation

⸻

Analytics System

Write Behind
+
Batch Flush

⸻

API Rate Limiting

Token Bucket
+
Redis Counter

⸻

Avoid Cache Stampede

Mutex Lock
OR
Soft TTL
OR
XFetch ⭐

⸻

📌 Golden Rules

🟦 Cache is NOT source of truth.

⸻

🟩 LRU is the default eviction policy.

⸻

🟨 Token Bucket is the default rate limiter.

⸻

🟥 Cache invalidation is harder than caching.

⸻

⭐ Aim for Cache Hit Ratio > 80%.

⸻

⭐ Protect databases from cache stampedes.

⸻

⭐ Redis Cluster is the standard scaling solution.

⸻

🚀 30-Second Revision

Cache Patterns:
Cache Aside ⭐
Write Through
Write Behind
Refresh Ahead
Eviction:
LRU ⭐
LFU
Stampede:
Lock
Soft TTL
XFetch ⭐
Rate Limiting:
Token Bucket ⭐
Sliding Window
Distributed:
Redis Cluster
Redis Counter
Target:
Cache Hit Ratio > 80%

