# Knowledge Base: Request Flow Internals
## Complete Backend Request Lifecycle

---

## Overview
This KB covers the complete journey of a request through a modern backend system, from client to database and back. Understanding this flow is fundamental to backend engineering, debugging, and system design.

---

## 1. REQUEST INITIATION: Client to Network

### 1.1 Browser/Client
- User initiates action (HTTP request)
- Request contains:
  - HTTP method (GET, POST, etc.)
  - URL
  - Headers
  - Body (if applicable)

### 1.2 DNS Resolution
**What happens:**
- Browser queries DNS server
- Domain name resolved to IP address
- Multiple DNS lookups might occur (cached vs. uncached)

**Critical insight:** 
Control plane ≠ Data plane
- DNS changes don't immediately break existing connections
- Failed DNS deployment doesn't mean traffic stops

**Production consideration:**
- DNS TTL affects how quickly changes propagate
- Route53 can fail on control plane while data plane continues serving

---

## 2. NETWORK LAYER: TCP/TLS

### 2.1 TCP Connection Establishment
```
Client                    Server
  |-------- SYN -------->|
  |<------ SYN-ACK ------|
  |-------- ACK -------->|
  [Connection established]
```

**What's happening:**
- Three-way handshake
- Establishes reliable communication channel
- Each side acknowledges receipt capability

### 2.2 TLS/SSL Handshake
```
Client                           Server
  |--- ClientHello (ciphers) --->|
  |<-- ServerHello + Cert ------|
  |--- Key Exchange + MAC ---->|
  |<------ Finished ----------|
  [Encrypted channel ready]
```

**Critical details:**
- Certificate validation
- Cipher suite negotiation
- Session key establishment

**Performance impact:**
- TLS handshake adds latency (especially first request)
- Connection reuse is critical
- Session resumption speeds up reconnections

---

## 3. LOAD BALANCER LAYER: ALB/NLB

### 3.1 Traffic Reception
**AWS Application Load Balancer (ALB):**
- Receives client connection
- Terminates TLS (optional)
- Inspects HTTP headers
- Decides target backend instance

### 3.2 Health Check Cycle
```
ALB
  |-- ping --> ECS Task 1 (healthy)
  |-- ping --> ECS Task 2 (healthy)
  |-- ping --> ECS Task 3 (unhealthy - removed from rotation)
```

**Important:** Health checks are independent of request handling
- Can be passing while application is degraded
- Should check actual business logic health, not just connectivity

### 3.3 Routing Decision
```
Routing Strategy:
  Path-based: /api/users → Backend A
              /api/products → Backend B
  Host-based: api.example.com → Backend A
              cdn.example.com → Backend B
  Header-based: x-client=premium → Backend A
                x-client=free → Backend B
```

---

## 4. COMPUTE LAYER: ECS/Kubernetes

### 4.1 Task/Pod Selection
**ECS (AWS):**
- Task: Unit of deployment
- Service: Manages multiple tasks
- Cluster: Shared resource pool

**What happens:**
- ALB routes to specific ECS task
- Network interface on task receives traffic
- Container starts processing

### 4.2 Container Networking
```
ALB → ENI (Elastic Network Interface)
    → Container Network Namespace
    → Application Process (Tomcat/Netty)
```

**Critical concept:**
- Container has isolated network namespace
- Port forwarding maps host port to container port
- Environment variables passed to container

---

## 5. APPLICATION SERVER: Tomcat/Jetty

### 5.1 Connection Acceptance
```
Tomcat Acceptor Thread
  |
  |-- Accepts connection from ALB
  |-- Places in incoming queue
  |-- Signals worker thread pool
```

### 5.2 Request Queue & Worker Threads
```
Request Queue
  ├─ Request 1 [waiting]
  ├─ Request 2 [waiting]
  ├─ Request 3 [waiting]
  └─ Request 4 [waiting]

Worker Thread Pool
  ├─ Thread 1 [processing Request A]
  ├─ Thread 2 [processing Request B]
  ├─ Thread 3 [blocked on DB]
  └─ Thread 4 [idle]
```

**Thread pool saturation scenario:**
```
All threads busy/blocked
  ↓
Incoming requests queue
  ↓
Queue reaches limit
  ↓
New requests rejected
  ↓
Latency spikes
  ↓
Client timeouts
  ↓
Retry storm
  ↓
Collapse
```

### 5.3 Important Distinction
**Main Thread:**
- Starts JVM
- Does NOT process requests

**Worker Threads (from pool):**
- Process incoming requests
- Handle HTTP parsing
- Pass to application

**Why this matters:**
- JVM stays alive while ANY worker thread exists
- Daemon threads can be interrupted
- Critical work should use user threads

---

## 6. SERVLET LAYER: DispatcherServlet

### 6.1 Request Reception
```
Worker Thread
  ↓
DispatcherServlet.doDispatch()
  ↓
[Request processing begins]
```

### 6.2 DispatcherServlet Flow
```
1. Receive HttpServletRequest
2. Determine Handler Mapping
   - URL → @RequestMapping
   - Method → @PostMapping, @GetMapping, etc.
3. Find Handler (Controller method)
4. Execute Handler Interceptors (preHandle)
5. Call handler method
6. Process return value (ModelAndView)
7. Execute Handler Interceptors (postHandle)
8. Render view / serialize response
9. Execute afterCompletion
10. Return response to client
```

---

## 7. SPRING FRAMEWORK LAYER

### 7.1 Controller Invocation
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        // Thread is here now
        // Request-scoped beans available
        // Spring context active
    }
}
```

### 7.2 Dependency Injection in Action
```
Spring Context
  ├─ ApplicationContext created at startup
  ├─ Beans instantiated and cached
  ├─ Dependencies resolved
  ├─ AOP proxies created
  └─ Interceptors ready

Request arrives:
  ↓
  Spring resolves dependencies for handler
  ↓
  All beans properly wired
  ↓
  Method executes
```

### 7.3 Request Scope
```
@Scope("request")
public class RequestScopedBean {
    // One instance per request
    // Shared across entire request processing
    // Destroyed when response sent
}
```

---

## 8. SERVICE & BUSINESS LOGIC LAYER

### 8.1 Call Chain
```
Controller
  ↓
Service (business logic)
  ↓
Repository (data access)
  ↓
Database
```

### 8.2 State at Service Level
```
Thread-local:
  - Authentication context
  - Request context
  - Correlation ID

Method parameters:
  - Passed data
  - Request details

Spring beans:
  - Autowired dependencies
  - Singletons (shared across requests)
  - Request-scoped beans (one per request)
```

---

## 9. DATA ACCESS LAYER: Repository/ORM

### 9.1 Connection Pooling
```
HikariCP Connection Pool
  ├─ Pool size: 10 connections
  ├─ Max idle: 5 connections
  ├─ Connection timeout: 30s
  
Request Flow:
  1. Borrow connection from pool
  2. Execute SQL
  3. Return connection to pool
```

### 9.2 Critical Concept: Connection Leaks
```
If connection NOT returned:
  ↓
Pool exhaustion
  ↓
New requests wait for timeout
  ↓
Entire application appears hung
  ↓
Cascading failure
```

**Real scenario:**
```
DB execution: 100ms
Thread sleeps after returning: 5 seconds
Connection still marked as "borrowed"
Pool saturates
All new requests hang
```

### 9.3 ORM Processing (Hibernate/JPA)
```
Service calls: repository.save(user)
  ↓
1. Hibernate intercepts
2. Lazy loading resolved (if needed)
3. Dirty checking occurs
4. SQL generated
5. Prepared statement created
6. Parameters bound
7. Query executed on DB
8. Results mapped back
9. Entity state updated
```

---

## 10. DATABASE LAYER

### 10.1 Query Execution
```
Database receives SQL
  ↓
Query parser validates syntax
  ↓
Query optimizer determines best plan
  ↓
Index selection
  ↓
Execution plan generated
  ↓
Query executes
  ↓
Results returned
```

### 10.2 Critical: Slow Query Impact
```
One slow query (5 second wait)
  ↓
Thread blocked, holding connection
  ↓
Connection unavailable to others
  ↓
Pool saturates slowly
  ↓
Latency increases for all requests
  ↓
Retry storm begins
  ↓
Complete collapse
```

### 10.3 Lock Contention
```
Transaction A locks row X
  ↓
Transaction B tries to update row X
  ↓
Transaction B waits
  ↓
Thread pool task blocked
  ↓
Cascade effect through application
```

---

## 11. RESPONSE PATH: Back to Client

### 11.1 Return Journey
```
Database returns result
  ↓
ORM maps to objects
  ↓
Service returns data
  ↓
Controller processes
  ↓
Spring serializes to JSON
  ↓
DispatcherServlet renders
  ↓
HTTP response created
  ↓
Sent to ALB
  ↓
ALB forwards to client
```

### 11.2 Serialization Overhead
```
Large result set (100k objects)
  ↓
Jackson serialization
  ↓
Convert to JSON string
  ↓
Network transmission
  ↓
Client parsing

If this is slow:
  ↓
Thread holds lock/connection longer
  ↓
Resource exhaustion
```

---

## 12. CRITICAL CONCEPTS FOR PRODUCTION

### 12.1 Dependency Chain Failures
```
Single slow dependency:

Slow DB
  ↓ (threads blocked)
Thread pool saturation
  ↓
Incoming queue grows
  ↓
Latency spikes
  ↓
Client timeouts
  ↓
Retries
  ↓
More load
  ↓
Complete failure
```

### 12.2 Resource Exhaustion Checklist
```
When system is slow, check:

1. CPU Usage?
   ✓ High → Compute bound (Reduce work)
   ✗ Low → Blocked waiting (Find blocker)

2. Thread Pool?
   ✓ Threads busy → Normal
   ✗ Threads busy but CPU low → I/O wait

3. Connection Pool?
   ✓ Available → Normal
   ✗ Exhausted → Query is slow

4. Memory?
   ✓ Stable → Normal
   ✗ Growing → Memory leak or cache explosion

5. Queue Depth?
   ✓ Small → Processing fast
   ✗ Large → Downstream slow
```

### 12.3 Thread Safety Implications
```
Singleton Bean (thread-safe required)
  ↓
Used by multiple worker threads simultaneously
  ↓
Shared mutable state = danger
  ↓
Must use synchronization or immutability

Request-scoped Bean (no thread safety needed)
  ↓
One per request
  ↓
No sharing with other threads
  ↓
No synchronization required
```

---

## 13. TIMEOUT & FAILURE SCENARIOS

### 13.1 Cascading Timeouts
```
Scenario: Upstream timeout = 10s, Downstream timeout = 10s

Request comes in
  ↓ (5s passes)
Calls slow downstream service
  ↓ (5s passes)
Downstream times out after 5s
  ↓
Returns error
  ↓
Back to upstream (total 10s)
  ↓
Upstream also times out
  ↓
Thread exhausted, connection held
  ↓
Resource leak
```

**Solution:** Upstream timeout < sum of downstream timeouts

### 13.2 Retry Storm
```
Initial failure
  ↓
Client retries (each request takes 30s)
  ↓
Multiple retry attempts in flight
  ↓
Load increases 3-4x
  ↓
Already-slow system gets more load
  ↓
Complete collapse

Example:
  100 concurrent requests × 3 retries = 300 concurrent requests
```

---

## 14. MONITORING THESE LAYERS

### 14.1 Key Metrics per Layer
```
DNS Layer:
  - DNS lookup time
  - Cache hit rate
  - Query count

Network Layer:
  - Connection count
  - Packet loss
  - Latency

Load Balancer:
  - Request count
  - Health check failures
  - Routing decisions

Tomcat:
  - Thread pool usage
  - Queue depth
  - Request count

Spring:
  - Request latency
  - Error rate
  - Method call duration

Repository:
  - Query time
  - Connection pool usage
  - N+1 query detection

Database:
  - Query execution time
  - Lock wait time
  - Slow query log
```

### 14.2 Tracing a Slow Request
```
Observe: Endpoint returning in 5 seconds (expected 500ms)

Debug steps:
1. Disable client-side retry logic → confirms server is slow
2. Check thread pool → if threads waiting, find what they wait on
3. Check DB connection pool → if exhausted, DB is bottleneck
4. Check slow query log → find specific slow query
5. Check query execution plan → identify index issue
6. Add database index
7. Retest → confirm improvement
```

---

## 15. ARCHITECTURE PATTERNS & IMPLICATIONS

### 15.1 Synchronous Request-Response
```
Advantages:
  ✓ Simple to reason about
  ✓ Easy to debug
  ✓ Straightforward error handling

Disadvantages:
  ✗ Thread per request
  ✗ Resources exhausted under load
  ✗ Blocking waits waste threads
```

### 15.2 Asynchronous/Reactive Patterns
```
Traditional (blocking):
Worker Thread
  ├─ [5s compute]
  ├─ [5s DB wait] ← wasting thread
  ├─ [1s serialize]
  └─ Return (1 thread for 11s)

Reactive (non-blocking):
Worker Thread
  ├─ [5s compute]
  ├─ Schedule DB work → releases thread
  ├─ ← thread handles other requests
  ├─ [DB returns] → resumes pipeline
  ├─ [1s serialize]
  └─ Return (1 thread shared across many requests)
```

---

## 16. COMMON PRODUCTION INCIDENTS

### 16.1 Database Slow → Application Hangs
```
Symptoms:
  - API latency high (30s+)
  - Thread pool full
  - CPU low
  - Connection pool exhausted

Root cause:
  - DB query slow (missing index)
  - Lock contention
  - Resource exhaustion on DB

Solution:
  - Add index
  - Break large transaction
  - Add circuit breaker
```

### 16.2 Memory Leak → Gradual Failure
```
Symptoms:
  - Latency increases gradually
  - GC time increases
  - Memory usage climbs
  - Eventually OOM

Root cause:
  - Cache never evicted
  - Static collection keeps references
  - ThreadLocal not cleared
  - Listener not removed

Detection:
  - Heap dump analysis
  - GC log review
  - Retained object analysis
```

### 16.3 Thread Pool Exhaustion → Cascading Failure
```
Symptoms:
  - Suddenly all requests timeout
  - Queue depth high
  - Latency spikes
  - Some endpoints work (served from cache)

Root cause:
  - Downstream service slow
  - Connection pool leak
  - Infinite loop in handler

Solution:
  - Add timeout
  - Fix connection leak
  - Fix infinite loop
```

---

## 17. DESIGN DECISIONS AT EACH LAYER

### 17.1 Thread Pool Sizing
```
Formula (rough):
  Threads = (Core Count × 2) + Outbound Connections

Example (4 cores):
  - I/O intensive: 8-16 threads
  - CPU intensive: 4-8 threads
  - Mixed: 10-12 threads
```

### 17.2 Connection Pool Sizing
```
Formula:
  Connections ≈ (Thread Count × Avg Concurrent DBCalls) × 1.2

Example:
  - 10 threads
  - Each makes 2 concurrent calls
  - 10 × 2 × 1.2 = 24 connections
```

### 17.3 Timeout Strategy
```
Total Request Timeout: 30s
  ├─ Service A call: 8s timeout
  ├─ Service B call: 8s timeout
  ├─ Service C call: 8s timeout
  ├─ Buffer: 4s
  └─ Reserve: 2s (for parsing/processing)

Total: 8+8+8+4+2 = 30s
```

---

## 18. INTERVIEW PREPARATION

### 18.1 Common Questions
1. **"Walk me through a request from browser to database"**
   - Answer: Use the complete flow above
   - Mention connection pooling, thread pools, timeouts
   - Discuss failure scenarios

2. **"What happens if database is slow?"**
   - Answer: Thread pool exhaustion → cascade → failure
   - Explain connection pool saturation
   - Mention retry storms

3. **"How would you debug latency?"**
   - Answer: Check each layer (DNS, network, LB, app, DB)
   - Mention metrics to monitor
   - Describe tracing approach

4. **"What's the difference between throughput and latency?"**
   - Throughput: requests/second
   - Latency: time per request
   - Both matter for production

---

## 19. SUMMARY: The Mental Model

```
Request Flow = Pipeline

Client
  ↓ (DNS resolution)
Network
  ↓ (TCP/TLS handshake)
Load Balancer
  ↓ (routing decision)
Compute (ECS/K8s)
  ↓ (container scheduling)
Application Server (Tomcat)
  ↓ (thread from pool)
Spring Framework
  ↓ (dependency injection)
Business Logic
  ↓ (service layer)
Data Access
  ↓ (connection from pool)
Database
  ↓ (query execution)
  ↓ (results back)
Response Serialization
  ↓ (JSON conversion)
Back through LB
  ↓
Back to Client

Bottleneck = Pipeline breaks
Resource Exhaustion = Pipeline jams
Timeout = Request stuck too long
Failure = Pipeline doesn't recover
```

---

## 20. KEY TAKEAWAYS

1. **Every layer has limited resources** (threads, connections, memory)
2. **One slow layer blocks all downstream layers**
3. **Thread safety matters for shared singletons**
4. **Connection pooling is critical for scalability**
5. **Timeouts prevent cascading failures**
6. **Monitoring each layer is essential**
7. **Understanding the flow is prerequisite for debugging**
8. **Synchronous request-response has fundamental scaling limits**

---

*This KB is meant as a reference for production debugging, interview preparation, and architectural discussions.*
