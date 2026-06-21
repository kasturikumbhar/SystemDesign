Knowledge Base: Real-Time System Design (Uber, WhatsApp, Notifications)

Matching Systems, Messaging Architecture, Geolocation, Scalability & Production Patterns

⸻

PART 1: UBER SYSTEM DESIGN (RIDE HAILING PLATFORM)

⸻

1.1 High-Level Architecture

Rider App
   ↓
API Gateway
   ↓
Backend Services
 ├── User Service
 ├── Driver Service
 ├── Trip Service
 ├── Matching Service
 ├── Pricing Service
 └── Location Service
Real-time Layer:
  WebSockets / gRPC streams
Data Stores:
  Redis (hot location data)
  PostgreSQL (trips)
  Cassandra/DynamoDB (events)
Event Bus:
  Kafka / SQS / Pulsar

⸻

1.2 Core Problem: Real-Time Matching

Goal:
Match Rider ↔ Nearest Available Driver in < 2 seconds
Constraints:
  ✓ High scale (millions of requests)
  ✓ Real-time location updates (every 2–5 sec)
  ✓ Low latency matching
  ✓ Fair driver distribution

⸻

1.3 Geolocation System

Grid-Based Partitioning (Core Idea)

City divided into grids:
+----+----+----+
| G1 | G2 | G3 |
+----+----+----+
| G4 | G5 | G6 |
+----+----+----+
| G7 | G8 | G9 |
+----+----+----+
Driver:
  D1 → G5
  D2 → G5
  D3 → G6
Rider request in G5
→ Search only local grid + neighbors

Geo Indexing (Production version)

* Geohash
* QuadTree
* H3 (Uber’s actual system)

Geohash Example:
Location → "tdr1k"
Nearby drivers share prefix:
tdr1k1
tdr1k2
tdr1k3
Prefix matching = fast proximity search

⸻

1.4 Driver Location Streaming

Driver App sends location:
every 2–3 seconds:
  lat, long, speed, heading
Pipeline:
Driver App
   ↓
Kafka Topic: driver_location
   ↓
Stream Processor (Flink / Spark Streaming)
   ↓
Redis GEO Index

Redis stores:

GEOADD drivers:city 77.12 28.61 driver_1
GEOADD drivers:city 77.13 28.62 driver_2

Query:

GEORADIUS drivers:city 77.12 28.61 3km

⸻

1.5 Matching Algorithm

Step 1: Rider requests ride
  → location captured
Step 2: Fetch nearby drivers
  radius = 1–3 km initially
Step 3: Filter drivers
  ✓ available
  ✓ not on ride
  ✓ vehicle type match
Step 4: Ranking score
score =
  - distance (weight 50%)
  - driver rating (20%)
  - acceptance rate (10%)
  - ETA (20%)
Step 5: Offer to driver
Driver gets request → 15s timeout
If reject:
  → expand radius
  → retry matching

⸻

1.6 Surge Pricing System

Demand > Supply → price increases
Inputs:
  requests per minute
  available drivers
  cancellation rate
Formula:
surge = demand / supply
If:
  2.0 → 2x price
  3.0 → 3x price
System:
Pricing Service → real-time stream processing

⸻

1.7 Fault Tolerance

Problems:
Driver app disconnects
Network lag
Location delay
Solutions:
✓ last known location cache (Redis)
✓ heartbeat from drivers
✓ retry matching pipeline
✓ idempotent trip creation

⸻

PART 2: WHATSAPP / REAL-TIME MESSAGING SYSTEM

⸻

2.1 Core Architecture

User A
  ↓
WebSocket Connection
  ↓
Messaging Gateway
  ↓
Message Service
  ↓
Queue (Kafka / MQ)
  ↓
Delivery Service
  ↓
User B

⸻

2.2 WebSocket Layer

Why WebSockets?
HTTP:
  request → response (not persistent)
WebSocket:
  persistent connection
  full duplex (bi-directional)
Flow:
Client connects:
  ws://server/chat
Server keeps connection open:
A ↔ Server ↔ B

⸻

2.3 Message Flow

User A sends message:
1. Client → Server
2. Server assigns message_id
3. Store in DB (persistent)
4. Push to Kafka queue
5. Delivery service consumes
6. Check recipient status
If online:
  push via WebSocket
If offline:
  store in inbox (retry later)

⸻

2.4 Delivery Guarantees

States:
Sent:
  client → server
Delivered:
  server → recipient device
Read:
  recipient opened message
Mechanism:
ACK system:
A → server → B
B sends ACK back
server updates status

⸻

2.5 Offline Messaging

If user offline:
Message stored in:
Inbox DB
+
Queue buffer
When user reconnects:
Fetch missed messages
Replay in order (by timestamp)

⸻

2.6 Ordering Problem

Challenge:
Messages must appear in correct order
Solution:
Partition by conversation_id
Kafka partition key:
conversation_123 → same partition
Guarantees:
✓ order preserved per chat

⸻

2.7 Presence System

Online / Offline status
Heartbeat every 10–30 sec:
User active → Redis set ONLINE
No heartbeat → mark OFFLINE
Presence store:
Redis:
  user_id → status

⸻

2.8 Scaling Messaging

Scale techniques:
✓ horizontal WebSocket servers
✓ sticky sessions (load balancer)
✓ message sharding by user_id
✓ Kafka partition scaling

⸻

PART 3: NOTIFICATION SYSTEM (EMAIL, PUSH, SMS)

⸻

3.1 High-Level Architecture

Event Producer
   ↓
Event Bus (Kafka / SNS)
   ↓
Notification Service
   ↓
Queue (SQS)
   ↓
Channel Workers:
  ├── Email Worker
  ├── SMS Worker
  ├── Push Worker
  └── In-App Worker

⸻

3.2 Event-Driven Flow

Example: Order Placed
Order Service emits:
OrderCreatedEvent
   ↓
Event Bus
   ↓
Notification Service consumes
   ↓
Decides channels:
✓ Email confirmation
✓ SMS alert
✓ Push notification

⸻

3.3 Queueing Strategy

Why queue?
✓ retries
✓ rate limiting
✓ burst handling
Flow:
Event → Queue → Worker
If failure:
  retry with exponential backoff
If still failing:
  send to DLQ (dead letter queue)

⸻

3.4 Retry Mechanism

Retry logic:
Attempt 1 → fail
wait 1s
Attempt 2 → fail
wait 2s
Attempt 3 → fail
wait 4s
Max retries = 5
Then:
→ DLQ
→ manual inspection

⸻

3.5 Idempotency (Critical)

Problem:
Duplicate notifications
Solution:
idempotency_key = user_id + event_id
If already processed:
  skip
Ensures:
✓ no duplicate SMS
✓ no repeated emails

⸻

3.6 Fanout Strategy

One event → many channels
OrderCreated
   ↓
Email Service
   ↓
SMS Service
   ↓
Push Service
Parallel execution improves latency

⸻

3.7 Rate Limiting

Avoid spam / overload
Rules:
SMS:
  10 msg/sec per user
Email:
  1000/min per domain
Push:
  unlimited (soft limit)
Implemented using:
Token Bucket / Leaky Bucket

⸻

3.8 Delivery Tracking

Status pipeline:
Queued
↓
Sent
↓
Delivered
↓
Clicked (optional)
Tracked via:
callbacks + webhook responses

⸻

PART 4: COMMON DESIGN PATTERNS

⸻

4.1 Real-Time Systems Pattern

Client
 ↓
WebSocket / API Gateway
 ↓
Event Bus
 ↓
Stream Processor
 ↓
Storage + Cache

⸻

4.2 Event Sourcing Pattern

Instead of storing state:
Store events:
UserLoggedIn
MessageSent
RideRequested
State is rebuilt by replay

⸻

4.3 CQRS Pattern

Command Side:
  write operations
Query Side:
  read optimized models
Separated for scale

⸻

PART 5: SCALABILITY STRATEGIES

⸻

5.1 Horizontal Scaling

Add more servers:
WebSocket servers
Kafka partitions
DB replicas
Scale linearly

⸻

5.2 Partitioning Strategy

Uber:
  location-based partitioning
WhatsApp:
  user_id / conversation_id
Notifications:
  user_id / region-based

⸻

5.3 Caching Layer

Redis used for:
✓ driver locations
✓ online users
✓ recent messages

⸻

KEY TAKEAWAYS

1. Uber = geospatial indexing + real-time matching
2. WhatsApp = WebSockets + ordered message delivery
3. Notifications = event-driven + retry + idempotency
4. Kafka/SQS is backbone of all systems
5. Redis is critical for real-time speed
6. Partitioning strategy determines scalability
7. Idempotency is mandatory for reliability
8. WebSockets enable persistent real-time communication
9. Event-driven architecture decouples everything
10. Most complexity = consistency + ordering + retries

⸻

Real-time systems are all about tradeoffs between latency, consistency, and scale. Master event flow first, not tools.
