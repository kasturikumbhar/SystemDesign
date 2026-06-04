# Knowledge Base: Distributed Systems & Scalability
## Replication, Consensus, CAP Theorem, System Design Patterns & Production Patterns

---

## PART 1: DISTRIBUTED SYSTEMS FUNDAMENTALS

### 1.1 Why Distributed Systems?
```
Single Machine Limitations:

CPU:
  Max throughput: ~10k requests/sec per core
  Solution: Add more cores/machines

Memory:
  Max RAM: ~256GB per machine
  Solution: Distribute data across machines

Disk:
  Max IOPS: ~100k per SSD
  Solution: Shard data across multiple disks

Availability:
  One machine = single point of failure
  Solution: Replicate across multiple machines

These problems compound:
  Small app: 1 machine works
  Medium app: 10 machines needed
  Large app: 100s or 1000s machines

Challenges:
  Network latency: Not negligible anymore
  Partial failures: Some machines down, others up
  Consistency: Multiple copies of data
  Coordination: Hundreds of machines to coordinate
```

### 1.2 Failure Modes
```
Failures in Distributed Systems:

Machine Failure:
  Crash: Machine stops responding
  Recovery: Restart and rejoin
  
Network Failure:
  Partition: Machines can't communicate
  Asymmetric: One can send, other can't receive
  Degradation: Slow/unreliable connection
  
Byzantine Failure:
  Machine behaves unpredictably
  Sends wrong data
  Corrupts data
  
Clock Skew:
  Machines have different time
  Causes ordering problems
  
Cascading Failure:
  One failure triggers another
  Retry storm from unhealthy service
  Entire system collapses

These aren't theoretical:
  AWS: Multi-AZ partition (lost zone connectivity)
  Google: Software bug crashes cluster
  Twitter: Cascading failure from one service
```

---

## PART 2: REPLICATION

### 2.1 Replication Models
```
Leader-Follower (Master-Slave):

Leader:
  - Receives all writes
  - Processes transactions
  - Writes to local disk
  - Sends changes to followers

Follower:
  - Receives updates from leader
  - Applies changes (replay)
  - Serves read-only traffic
  - Cannot accept writes

Read Flow:
  Read request → Can go to Leader or any Follower
  ✓ Fast reads (distribute across replicas)
  ✓ Scale reads (add more followers)

Write Flow:
  Write request → Must go to Leader
  Leader processes
  Leader sends to Followers
  Followers apply

Benefits:
  ✓ High availability (failover)
  ✓ Read scaling
  ✓ Simple to understand
  
Drawbacks:
  ✗ All writes through leader (not scalable)
  ✗ Replication lag (followers behind)
  ✗ Split-brain if leader fails

Peer-to-Peer (Multi-Master):

Each node:
  - Can receive reads AND writes
  - Replicates to all other nodes
  - Conflict resolution needed

Write Flow:
  Write request → Any node accepts
  → Propagates to all others
  ✓ All nodes can write
  ✓ No single point of failure
  
Challenges:
  ✗ Concurrent writes to same data
  ✗ Conflicts must be resolved
  ✗ Consistency harder to guarantee

Example Conflict:
  Node A: user.age = 30
  Node B: user.age = 31 (same user, concurrent writes)
  
  Now both versions exist
  Solution needed:
    - Last-write-wins (keep 31)
    - Merge (more complex)
    - Application logic
    
Used by: DynamoDB, Cassandra (Eventual Consistency)
```

### 2.2 Replication Lag
```
Scenario: Leader-Follower setup

Leader receives write:
  1. Process write
  2. Commit locally (0ms)
  3. Send to Follower (10ms network latency)
  4. Follower receives (10ms total)
  5. Follower applies (12ms total)

During this 12ms window:
  ✓ Reads from Leader: See latest data
  ✗ Reads from Follower: See old data

Problems This Causes:

1. Read-After-Write Inconsistency
   User writes profile: name = "John"
   Immediately reads from follower: name = "" (old)
   User confused!
   
   Solution: Route recent writes to leader

2. Monotonic Read Violation
   Read 1 from Follower A: data_version = 5
   Follower A restarted, replication lagged
   Read 2 from Follower A: data_version = 3
   Version went backwards!
   User confused!
   
   Solution: Sticky reads (same client → same server)

3. Causal Consistency Violation
   T1: Write x = 5
   T2: Write y = x + 10 (y = 15)
   Read y from old follower: y = null (x not replicated yet)
   Invalid!
   
   Solution: Wait for replication before reading

Replication Lag in Practice:
  Microseconds: Negligible
  Milliseconds: Noticeable
  Seconds: Serious problem
  
Monitoring:
  Measure replication delay
  Alert if > threshold
  Consider degrading service
```

---

## PART 3: CONSENSUS & CAP THEOREM

### 3.1 CAP Theorem
```
Three Properties:

Consistency (C):
  All nodes see same data
  Reading from any node returns latest value
  
  Guarantee: Strong consistency
  Tradeoff: Must coordinate (slow)

Availability (A):
  System always responds
  No failures → always responding
  
  Guarantee: Responsive
  Tradeoff: Might return stale data

Partition Tolerance (P):
  System survives network partition
  Nodes cut off from each other
  System continues operating
  
  Guarantee: Fault tolerance
  Tradeoff: Can't maintain both C and A

CAP Theorem:
  In presence of network partition
  Choose at most TWO of {C, A, P}

In Reality:
  Network partitions WILL happen
  Therefore P is mandatory
  
Choice becomes:
  CP: Consistency + Partition tolerance
      (sacrifice Availability)
      
      Example: PostgreSQL (single leader)
      Leader fails → No writes until recovery
      Read-only mode during partition
      
      ✓ Strong consistency
      ✗ Not always available for writes
      
  AP: Availability + Partition tolerance
      (sacrifice Consistency)
      
      Example: DynamoDB, Cassandra
      Both sides accept writes during partition
      Conflict resolution later
      
      ✓ Always available
      ✗ Eventual consistency (stale data temporarily)

Timeline of CAP:

  Normal (no partition):
    Can appear to have all three
    High consistency + high availability
    
  During partition:
    MUST choose: consistency OR availability
    
  After partition:
    System heals
    Consistency restored (eventually)

In Practice:
  Most systems: AP (eventual consistency)
  High-volume apps favor availability
  Data loss rare (replicated)
  Temporary inconsistency acceptable
  
  Critical apps: CP (strong consistency)
  Financial systems
  Banking
  Inventory management
```

### 3.2 Consensus Algorithms
```
Problem:
  Distributed nodes need to agree on value
  Despite failures
  Despite delays
  Despite partitions

Solution: Consensus Algorithm

Raft (Simpler, used by Consul):

Roles:
  Leader: Accepts writes, commands
  Follower: Replicates leader's log
  Candidate: Competing to be leader

Three sub-problems:

1. Leader Election
   No leader elected
   ↓
   Followers wait random timeout (150-300ms)
   ↓
   First timeout → Become candidate
   ↓
   Ask others to vote
   ↓
   If get majority → Become leader
   ↓
   Append entries to prove alive
   
   Ensures: One leader at a time

2. Log Replication
   Client sends command to leader
   ↓
   Leader adds to log (uncommitted)
   ↓
   Sends to followers
   ↓
   Followers append to log
   ↓
   Followers acknowledge
   ↓
   Leader commits (when majority ack)
   ↓
   Leader applies to state machine
   ↓
   Sends commit message to followers
   
   Ensures: All nodes execute same sequence

3. Safety
   Ensure no data loss
   
   Rules:
   - Only leader can append entries
   - Followers accept from leader only
   - Entries must be committed before applying
   - If candidate becomes leader:
     Previous uncommitted entries discarded

Example: 5-node cluster, 1 fails

  Node 1 (Leader)
  Node 2 (Follower)
  Node 3 (Follower)
  Node 4 (Follower)
  Node 5 (Down)
  
  Write request to leader
  ↓
  Leader appends to log: [entry 5]
  ↓
  Sends to followers 2,3,4
  ↓
  3 acknowledge (2,3,4)
  ↓
  Majority (3 of 5)
  ↓
  Leader commits
  ↓
  Tolerates: 1 node failure (need 3 of 5)
  
  If leader fails:
  ↓
  Followers wait timeout
  ↓
  One becomes candidate
  ↓
  Gets votes from 2 others
  ↓
  Becomes new leader
  ↓
  System continues

Used by: Consul, etcd (K8s), Zookeeper
```

---

## PART 4: SHARDING & PARTITIONING

### 4.1 Sharding Strategies
```
Problem: Too much data for one machine

Data Growth:
  Year 1: 100GB (fits on one machine)
  Year 2: 1TB (fits on one machine)
  Year 3: 10TB (doesn't fit!)
  
Solution: Shard across multiple machines

Sharding Strategy 1: Range-based
  
  Shard 1: user_id 1-1M
  Shard 2: user_id 1M-2M
  Shard 3: user_id 2M-3M
  
  Routing:
    user_id = 500k → Shard 1
    user_id = 1.5M → Shard 2
    user_id = 2.5M → Shard 3
  
  Pros:
    ✓ Simple routing
    ✓ Range queries efficient (all data in one shard)
    
  Cons:
    ✗ Uneven distribution (popular ranges hot)
    ✗ Rebalancing complex

Sharding Strategy 2: Hash-based
  
  hash(user_id) % 3
    hash(500k) % 3 = 0 → Shard 1
    hash(1.5M) % 3 = 1 → Shard 2
    hash(2.5M) % 3 = 2 → Shard 3
  
  Pros:
    ✓ Even distribution
    ✓ Predictable routing
    
  Cons:
    ✗ Rebalancing causes redistribution
    ✗ Range queries require hitting all shards

Sharding Strategy 3: Directory-based
  
  Directory service:
    user_id → shard mapping
    500k → Shard 1
    1.5M → Shard 2
    2.5M → Shard 3
  
  Lookup: Ask directory → Go to shard
  
  Pros:
    ✓ Flexible
    ✓ Easy rebalancing
    
  Cons:
    ✗ Directory lookup overhead
    ✗ Directory becomes bottleneck

Sharding Strategy 4: Consistent Hashing
  
  Ring of 360 positions
  
  Shards: A, B, C at positions 0, 120, 240
  
  Key hashes to position
  → Find next shard clockwise
  
  Pros:
    ✓ Minimal rebalancing when adding shard
    ✓ Load balanced
    
  Cons:
    ✗ Slightly more complex
    
  When adding Shard D:
    Only keys between C and D move
    Other keys stay (vs redistribution with mod)

Example: DynamoDB uses Consistent Hashing + Virtual Nodes
```

### 4.2 Hot Shard Problem
```
Problem: Uneven load distribution

Scenario:
  Shard 1: Celebrity user (100M followers)
    Load: 1M reads/sec
    
  Shard 2: Regular users (million users)
    Load: 10k reads/sec
    
  Shard 3: Regular users
    Load: 10k reads/sec

Result:
  Shard 1 saturated
  Others idle
  Bottleneck at Shard 1

Solutions:

1. Sub-sharding
   Shard 1 too hot → Split into Shard 1a, 1b, 1c
   Distribute hot data across multiple physical shards
   
   Routing: hash(user_id, sub_shard) % 3
   
   Cons: Complex routing logic

2. Caching
   Cache hot data aggressively
   Most requests hit cache
   Only overflow hits database
   
   Example: Redis cache for celebrity posts
   
   Pros: Simple, effective
   Cons: Cache miss still slow

3. Read Replicas
   Replicate hot shard to multiple replicas
   Distribute reads across replicas
   
   Pros: Spreads read load
   Cons: Still one write destination

4. Denormalization
   Pre-compute aggregates
   Avoid expensive queries on hot data
   
   Example: Pre-compute follower counts
            Update asynchronously

Prevention:
  Monitor shard utilization
  Rebalance early
  Split before saturation
```

---

## PART 5: EVENTUAL CONSISTENCY

### 5.1 Eventual Consistency Model
```
Guarantee:
  "If no new writes after time T,
   all nodes will eventually see same data"

Not:
  "All nodes see same data immediately"

Timeline:

T=0:
  Write x = 5 to Node A
  
T=5ms:
  Node A has: x = 5
  Node B has: x = 0 (old)
  Node C has: x = 0 (old)
  (Inconsistent)
  
T=100ms:
  Node A has: x = 5
  Node B has: x = 5 (replicated)
  Node C has: x = 0 (replication in progress)
  
T=200ms:
  Node A has: x = 5
  Node B has: x = 5
  Node C has: x = 5 (fully replicated)
  (Consistent!)
  
Guarantee: Eventually consistent

Trade-offs:

For User:
  ✓ High availability (no waiting for consensus)
  ✓ Low latency (can return immediately)
  
  ✗ Might see stale data temporarily
  ✗ Need conflict resolution

For System:
  ✓ Better scalability (no coordination overhead)
  ✓ Partition-tolerant (works during splits)
  
  ✗ Complex application logic (handle stale data)
  ✗ Debugging harder (timing-dependent bugs)

When to use:
  ✓ Social media feeds (stale OK)
  ✓ Analytics (eventual accuracy fine)
  ✓ Caching (stale fine)
  
  ✗ Financial transactions (need strong consistency)
  ✗ Inventory (need accurate counts)
  ✗ User authentication (need fresh state)
```

### 5.2 Conflict Resolution
```
Scenario: Multi-master replication

Node A writes: user.age = 30
Node B writes: user.age = 31 (same user, concurrent)

Both think they're correct
Now both changes propagate to each other

Conflict:
  Which is the "true" value?

Solutions:

1. Last-Write-Wins (LWW)
   Keep write with latest timestamp
   Problem: Relies on clock synchronization
   
   If Node A's clock ahead:
     A: timestamp 100
     B: timestamp 99
     → Keep A's value even if B wrote later
     → Wrong!

2. Vector Clocks
   Each node tracks causality
   
   Node A: [A:5, B:3] (seen 5 from A, 3 from B)
   Node B: [A:3, B:4]
   
   If A sees [A:5, B:3] and then [A:3, B:4]:
     A's clock is ahead → A's write happened after
     → Keep A's value
   
   If neither is strictly ahead:
     Concurrent writes → Conflict

3. Application-Defined Resolution
   App knows domain better
   
   Example: Merge function
   user.age: merge(30, 31) → 31 (keep max)
   user.name: merge("John", "Jon") → Keep both? Merge?
   
   More complex but semantically correct

4. Conflict-Free Replicated Data Types (CRDTs)
   Data structure designed for concurrent updates
   No conflicts possible by design
   
   Example: Counter CRDT
     Node A: +5
     Node B: +3
     → Both agree on +8 (commutative)
   
   Example: Set CRDT
     Node A: add X
     Node B: add Y
     → Both agree on {X, Y}
   
   Pros: No conflict resolution needed
   Cons: Limited to certain data types

Real-world:
  DynamoDB: LWW (but hooks for custom)
  Cassandra: LWW + tuple timestamps
  Git: User-defined (conflicts in working tree)
  CouchDB: CRDT-inspired approach
```

---

## PART 6: MONITORING DISTRIBUTED SYSTEMS

### 6.1 Observability
```
Three Pillars:

Metrics (What):
  Counters: Requests, errors, bytes sent
  Gauges: Active connections, queue depth
  Histograms: Latency distribution, response sizes
  
  Tools: Prometheus, Graphite, CloudWatch
  
  Example:
    request_count: 100k
    error_rate: 0.1% (100 errors)
    p99_latency: 500ms (99% under 500ms)

Logs (Why):
  Detailed record of events
  Searchable, contextualized
  
  Tools: ELK (Elasticsearch, Logstash, Kibana), Splunk, Loki
  
  Example:
    2024-01-15 10:23:45.123
    ERROR [OrderService]
    Failed to process order: order_id=12345
    reason: timeout calling payment_service
    duration: 30000ms

Traces (How):
  Follow request through system
  Understand call flow and dependencies
  
  Tools: Jaeger, Zipkin, X-Ray
  
  Example:
    Trace ID: abc-123
    Span 1: API Gateway (2ms)
      → Span 2: OrderService (10ms)
        → Span 3: PaymentService (8ms - TIMEOUT)
        → Span 4: NotificationService (5ms)
      → Span 5: LogService (1ms)
    
    Identifies: PaymentService timeout is culprit

Unified:
  Metrics tell WHAT (rate elevated)
  Logs tell WHY (error in service X)
  Traces tell HOW (request flow through system)
```

### 6.2 Alerting Strategy
```
Alert when:
  Service down (availability)
  Error rate high
  Latency high
  Database replication lag
  Disk space low
  Memory leak (growing over time)

Alert severity:

Page (Immediate):
  Users directly impacted
  Service unavailable
  Data loss risk
  
  Example: "API returning 500 errors"

On-Call Rotation:
  Someone on-call 24/7
  Pager goes off
  Must respond within 5-15 min

Ticket (Business Hours):
  Degraded but not critical
  Users inconvenienced
  
  Example: "API latency p99 = 1s"

Context (No Alert):
  Informational
  Investigate but not urgent
  
  Example: "New deploy finished"
```

---

## KEY TAKEAWAYS

1. **Replication** enables high availability (failover)
2. **Replication lag** causes consistency issues (eventual)
3. **CAP theorem** forces choice (P mandatory, C or A)
4. **Consensus** (Raft) enables agreement despite failures
5. **Sharding** enables horizontal scaling (but adds complexity)
6. **Hot shards** become bottlenecks (monitor, split early)
7. **Eventual consistency** trades immediacy for availability
8. **Conflict resolution** needed in multi-master (LWW, CRDTs)
9. **Monitoring** (metrics, logs, traces) essential for debugging
10. **Distributed systems hard** - start simple, add complexity

---

*Design for scale, prepare for failure, monitor everything.*
