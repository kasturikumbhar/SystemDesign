🚀 Kafka & Event Streaming

Notion-Style System Design Handbook

⸻

📑 Table of Contents

1. Why Message Queues?
2. Kafka Architecture
3. Partitions & Scaling
4. Consumer Groups
5. Delivery Guarantees
6. Error Handling & DLQ
7. Event Sourcing
8. Production Architecture
9. Interview Cheat Sheet
10. Golden Rules

⸻

⚡ 1. Why Message Queues?

⸻

❌ Without Kafka

Order Service
     │
     ├── Email Service
     ├── Inventory Service
     ├── Analytics Service
     └── Notification Service

Problems:

🔴 Synchronous

🔴 Tight coupling

🔴 Cascading failures

🔴 Retry complexity

🔴 Service downtime affects others

⸻

✅ With Kafka

Order Service
      │
      ▼
Kafka Topic (OrderCreated)
      │
 ┌────┼────┬────┐
 ▼    ▼    ▼    ▼
Email Inventory Analytics Notification

Benefits:

🟢 Asynchronous

🟢 Decoupled

🟢 Fault tolerant

🟢 Scalable

🟢 Durable

⸻

⭐ Kafka turns services into event producers and consumers.

⸻

🏗 2. Kafka Core Concepts

⸻

Topic

A stream of messages.

Examples:

orders
payments
users
notifications

⸻

Partition

A topic is divided into partitions.

Orders Topic
Partition 0
[msg1][msg2][msg3]
Partition 1
[msg4][msg5][msg6]
Partition 2
[msg7][msg8][msg9]

Benefits:

✅ Parallelism

✅ Scalability

✅ Ordering within partition

⸻

Broker

Kafka server.

Example cluster:

Broker 1
Broker 2
Broker 3

Partitions are distributed among brokers.

⸻

Offset

Unique position inside a partition.

Partition 0
Offset 0 → Order1
Offset 1 → Order2
Offset 2 → Order3
Offset 3 → Order4

Consumers remember:

Current offset = 2
Next read starts from 3

⸻

Consumer Group

Multiple consumers cooperate.

Orders Topic
(3 partitions)
Consumer Group
Consumer 1 → Partition 0
Consumer 2 → Partition 1
Consumer 3 → Partition 2

No duplicates inside same group.

⸻

🚀 Message Flow

⸻

Producer Side

Producer
    │
send()
    │
Partitioner
    │
Leader Broker
    │
Write Log
    │
Replicas
    │
ACK

⸻

Consumer Side

poll()
    │
Receive Messages
    │
Process
    │
Commit Offset
    │
Next Poll

⸻

📦 3. Partitioning Strategies

⸻

🟦 Key-Based Partitioning ⭐

Same key → same partition.

user:123 → Partition 0
user:456 → Partition 1
user:123 → Partition 0

Guarantees:

✅ Ordering

⸻

Use Cases

🏦 Banking

🛒 Orders

👤 User activity

⸻

🟨 Round Robin

Message1 → P0
Message2 → P1
Message3 → P2
Message4 → P0

Benefits:

🟢 Balanced load

Downside:

🔴 No ordering guarantee

⸻

Use Cases

Analytics

Logs

Telemetry

⸻

🟩 Time Partitioning

logs-2025-01
logs-2025-02
logs-2025-03

Great for retention.

⸻

🎯 Partitioning Decision

Need ordering?
      ↓
YES
 ↓
Key-based ⭐
Need throughput?
      ↓
YES
 ↓
Round Robin
Time-series?
      ↓
YES
 ↓
Time Partitioning

⸻

🔒 4. Replication & Durability

Replication Factor = 3

Broker 1 (Leader)
[msg1]
[msg2]
[msg3]
      ↓
Broker 2 (Replica)
[msg1]
[msg2]
[msg3]
      ↓
Broker 3 (Replica)
[msg1]
[msg2]
[msg3]

⸻

Write Path

Producer
     │
Leader Broker
     │
Replica 1
     │
Replica 2
     │
ACK Producer

⸻

Broker Failure

Before:

Broker1 = Leader

Crash!

Broker2 promoted to Leader

No message loss.

⸻

⭐ Replication Factor = 3 is industry standard.

⸻

👥 5. Consumer Groups

Topic:

3 partitions

Consumers:

Consumer1
Consumer2
Consumer3

Assignments:

C1 → P0
C2 → P1
C3 → P2

⸻

Rebalancing

New consumer joins:

Consumer4 joins
      │
Coordinator detects
      │
Rebalance
      │
Partitions reassigned

Temporary pause:

3-10 sec.

⸻

Graceful Shutdown

SIGTERM
     │
Commit Offset
     │
Leave Group
     │
Rebalance

No message loss.

⸻

Scaling Rule

Consumers ≤ Partitions

Example:

10 partitions
15 consumers

Only:

10 active
5 idle

⸻

⭐ Partition count limits parallelism.

⸻

📬 6. Delivery Guarantees

⸻

🔴 At-Most-Once

Process Message
      │
Crash
      │
Message Lost

Fast but unsafe.

Use Cases

Analytics

Metrics

Logs

⸻

🟡 At-Least-Once ⭐

Process
     │
Crash before commit
     │
Reprocess

Possible duplicate.

No data loss.

⸻

Solution

Idempotency

if email_sent(orderId):
    return

⸻

Use Cases

Payments

Orders

Transactions

⸻

🟢 Exactly Once

Guarantees:

No Loss
No Duplicate

Requires:

Replication
+
Transactions
+
Atomic Commit
+
Idempotency

Complex and slower.

⸻

⭐ Most companies use:

At-Least-Once + Idempotency

⸻

⚠ 7. Dead Letter Queue (DLQ)

Normal Flow

Payments Topic
      │
Consumer
      │
Success

Failure:

Payments Topic
      │
Consumer
      │
Error
      │
DLQ Topic

⸻

Retry Pattern

1 sec
2 sec
4 sec
8 sec
16 sec

(Exponential Backoff)

⸻

Permanent Errors

Immediately send to:

payments-dlq

⸻

Use Cases

Invalid JSON

Bad schema

Payment failures

External API failures

⸻

🔥 8. Event Sourcing

Traditional DB:

Balance = $900

Old history lost.

⸻

Event Sourcing

AccountCreated
      ↓
Deposit $1000
      ↓
Transfer $100
      ↓
Deposit $500

Replay events:

Current Balance = $1400

⸻

Benefits:

✅ Complete audit trail

✅ Replay history

✅ Time travel

✅ Compliance

⸻

Snapshot Optimization

Instead of replaying 1M events:

Snapshot
Balance = $1400
Replay last 100 events

Much faster.

⸻

Use Cases

🏦 Banking

📈 Trading

🛒 Orders

Saga Pattern

⸻

🏗 Production Architecture

              Producers
                  │
                  ▼
            Kafka Cluster
        ┌──────┬──────┬──────┐
        ▼      ▼      ▼
     Broker1 Broker2 Broker3
        ▲      ▲      ▲
        └────Replication────┘
                  │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
Email Group  Inventory Group Analytics Group
      │          │             │
     Redis      MySQL       ElasticSearch

⸻

🌎 Event-Driven Microservices

User Places Order
       │
       ▼
Order Service
       │
OrderCreated Event
       │
       ▼
Kafka Topic
 ┌─────┼─────┬─────┐
 ▼     ▼     ▼     ▼
Payment
Inventory
Notification
Analytics

Everything is decoupled.

⸻

🎯 Interview Cheat Sheet

⸻

Order System

Order Service
      ↓
Kafka
      ↓
Payment Service
Inventory Service
Email Service
Analytics Service

⸻

Need Ordering?

Use Key-Based Partitioning ⭐

⸻

Need Scalability?

More Partitions
+
More Consumers

⸻

Need Reliability?

Replication Factor = 3

⸻

Need Fault Tolerance?

At-Least-Once
+
Idempotency ⭐

⸻

Failed Messages?

DLQ
+
Exponential Backoff

⸻

Audit Trail?

Event Sourcing
+
Snapshots

⸻

⭐ Golden Rules

⸻

🟦 Partition = Unit of Parallelism

⸻

🟩 Ordering is guaranteed only inside a partition.

⸻

🟨 Consumers cannot exceed partitions.

⸻

🟥 Rebalancing pauses processing.

⸻

⭐ Replication Factor = 3.

⸻

⭐ At-Least-Once + Idempotency is the practical default.

⸻

⭐ Dead Letter Queues are mandatory in production.

⸻

⭐ Event sourcing provides complete history.

⸻

🚀 30-Second Revision

Kafka = Topic + Partition + Broker
Partition = Scalability
Consumer Group = Parallelism
Offset = Position
Replication = Durability
At-Least-Once + Idempotency ⭐
DLQ for failures
Event Sourcing = Audit Trail
Consumers ≤ Partitions
Ordering guaranteed only inside partition

This is very close to a FAANG/L6-L7 System Design handbook format. The next natural chapter after Kafka would be Databases (SQL vs NoSQL, Indexes, Sharding, Replication, CAP, Consistency, Transactions), which ties directly into system design interviews.
