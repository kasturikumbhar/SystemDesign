# Knowledge Base: Multithreading & Concurrency
## Deep Understanding of Thread Safety, Synchronization & Production Issues

---

## SECTION 1: FUNDAMENTALS

### 1.1 What is a Thread?
```
Process: Running instance of a program
  ├─ Has its own memory space
  ├─ Has its own file descriptors
  ├─ Heavyweight
  
Thread: Execution context within a process
  ├─ Shares memory space with other threads
  ├─ Lightweight compared to process
  ├─ Can access same objects
  └─ **This creates concurrency problems**
```

**Critical insight:**
Multiple threads in same process can simultaneously access same objects.
This requires synchronization.

### 1.2 Thread Creation
```java
// Option 1: Extend Thread
class MyThread extends Thread {
    public void run() { /* work */ }
}
MyThread t = new MyThread();
t.start();

// Option 2: Implement Runnable
class MyRunnable implements Runnable {
    public void run() { /* work */ }
}
Thread t = new Thread(new MyRunnable());
t.start();

// Critical: Always call start(), NOT run()
// start() creates new thread
// run() executes in current thread
```

### 1.3 Thread Lifecycle
```
NEW
  ↓ (start() called)
RUNNABLE
  ↓ (scheduler assigns CPU)
RUNNING
  ↓ (yield / thread.wait() / synchronized block)
WAITING / BLOCKED
  ↓ (notify / lock released)
RUNNABLE
  ↓ (run() completes)
TERMINATED
```

---

## SECTION 2: THE CONCURRENCY PROBLEM

### 2.1 Shared Mutable State
```
Scenario: Two threads, one variable

int count = 0;

Thread 1: count++  // read → increment → write
Thread 2: count++  // read → increment → write

Expected: count = 2
Actual: count = 1 (sometimes)

Why?
Thread 1: read(0) → increment(1) → write(1)
Thread 2: read(0) → increment(1) → write(1)  ← read stale value!

Result: count = 1
```

### 2.2 Race Condition
```
Definition: 
Multiple threads racing to modify shared state
Result depends on execution order (unpredictable)

Characteristics:
  - Hard to reproduce
  - Non-deterministic
  - Fails under specific timing
  - Often appears under load
  - Disappears with debugging
```

### 2.3 Memory Visibility
```
Scenario: Flag update not seen by other thread

Thread 1:
  shutdown = true  // write to variable
  
Thread 2:
  while(!shutdown) {  // keeps reading old value = false
    doWork();
  }

Why?
- Thread 1's write cached in CPU cache
- Thread 2's read from its own CPU cache
- Changes not visible across cores

Solution: Use volatile keyword
```

---

## SECTION 3: SYNCHRONIZATION PRIMITIVES

### 3.1 synchronized Keyword

#### How it works:
```
Every object has intrinsic lock (monitor)
synchronized block = obtain lock → execute → release lock
```

#### Synchronized Instance Method
```java
public synchronized void increment() {
    count++;  // Only one thread at a time
}

// Equivalent to:
public void increment() {
    synchronized(this) {
        count++;
    }
}
```

#### Synchronized Static Method
```java
public synchronized static void doWork() {
    // Lock on Class object, not instance
}

// Equivalent to:
public static void doWork() {
    synchronized(MyClass.class) {
        // ...
    }
}
```

#### Synchronized Block
```java
public void transfer(Account from, Account to, int amount) {
    synchronized(from) {  // Lock from account
        from.withdraw(amount);
    }
    synchronized(to) {    // Lock to account
        to.deposit(amount);
    }
}

// PROBLEM: Deadlock possible
// Thread 1: locks A, tries to lock B
// Thread 2: locks B, tries to lock A
// Both wait forever
```

### 3.2 Mutual Exclusion
```
Definition: Only one thread in critical section

Critical Section:
  count++
  
With synchronization:
  Thread 1: [enter] count++ [exit]
  Thread 2: [wait] ---- [enter] count++ [exit]
  Thread 3: [wait] ---- [wait] ---- [enter] count++ [exit]

Result: count = 3 (correct)
```

### 3.3 Visibility Guarantee
```
synchronized ensures:
  1. Mutual exclusion (one thread at a time)
  2. Memory visibility (changes visible to all threads)

Memory barrier:
  - Entry to synchronized: load latest values from memory
  - Exit from synchronized: flush writes back to memory
```

---

## SECTION 4: ATOMIC OPERATIONS

### 4.1 Compare-And-Swap (CAS)
```java
// Atomic operation: check and update
if(current == expected) {
    current = new;  // Only if current still equals expected
} else {
    retry;
}

// Example: AtomicInteger
AtomicInteger counter = new AtomicInteger(0);

// Atomic increment without synchronized
counter.incrementAndGet();  // count++ atomically
```

### 4.2 CAS Loop Pattern
```java
AtomicInteger count = new AtomicInteger(0);

// Without CAS (synchronized):
synchronized(count) {
    int old = count.get();
    count.set(old + 1);
}

// With CAS (lock-free):
while(true) {
    int current = count.get();
    int next = current + 1;
    if(count.compareAndSet(current, next)) {
        break;  // Success
    }
    // Retry (current changed)
}
```

### 4.3 ABA Problem
```
Scenario: Value changes but looks the same

Thread 1:
  value = A (read)
  prepare: newValue = B
  
Thread 2:
  value = A → B → A (changed and changed back)

Thread 1:
  CAS(A, B) succeeds  // Doesn't realize value changed!

Solution: AtomicStampedReference
  - Store both value and version
  - CAS checks both value AND version
```

---

## SECTION 5: THREAD-SAFE COLLECTIONS

### 5.1 HashMap Problem
```
HashMap NOT thread-safe

Concurrent modification scenario:
Thread 1: Add 100 items to HashMap
Thread 2: Iterate over HashMap (for-each loop)

Possible outcomes:
  1. ConcurrentModificationException
  2. Infinite loop (during resize)
  3. Segmentation fault
  4. Data loss
```

### 5.2 Collections.synchronizedMap
```java
Map<String, String> map = Collections.synchronizedMap(
    new HashMap<>()
);

// Each operation synchronized
map.put("key", "value");  // Synchronized on map
map.get("key");           // Synchronized on map
```

**Problem:** Compound operations still not atomic
```java
if(!map.containsKey("key")) {
    map.put("key", value);  // Race condition here!
}

// Another thread can insert between containsKey and put
```

### 5.3 ConcurrentHashMap
```java
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();

// Reduces lock contention
map.put("key", "value");  // Lock only this bucket
map.get("key");           // Lock only this bucket

// Multiple threads can read different buckets simultaneously
```

**Advantages:**
- Segment-based locking
- Better scalability
- Lower contention than synchronizedMap
- Still thread-safe

---

## SECTION 6: THREADLOCAL

### 6.1 Problem ThreadLocal Solves
```
Scenario: Object needs per-thread state

Without ThreadLocal:
  Thread 1: creates Connection
  Thread 2: creates Connection
  Both share same Connection object?
  NO! Need separate instance per thread

Solution: ThreadLocal
```

### 6.2 ThreadLocal Usage
```java
ThreadLocal<Connection> connHolder = new ThreadLocal<>();

// Each thread gets own instance
Thread 1: connHolder.set(conn1);
Thread 2: connHolder.set(conn2);

Thread 1: connHolder.get();  // Returns conn1
Thread 2: connHolder.get();  // Returns conn2

// No synchronization needed
```

### 6.3 ThreadLocal Leak
```
Scenario: ThreadLocal not cleaned up

Thread from pool:
  1. Request arrives
  2. connHolder.set(connection)
  3. Process request
  4. Complete (but forget remove()!)
  5. Thread returned to pool
  6. New request comes
  7. connHolder.get() returns old connection!

Solution:
  try {
    connHolder.set(conn);
    // work
  } finally {
    connHolder.remove();  // Always clean up!
  }
```

---

## SECTION 7: LOCKS & LOCK CONTENTION

### 7.1 Lock Contention
```
Lock Contention = Threads waiting for same lock

Scenario 1: Low Contention
  Thread 1: [Lock] ← quick
  Thread 2: [Wait] [Lock] ← quick
  Result: Both complete quickly

Scenario 2: High Contention
  Thread 1: [Lock] ← takes 1s
  Thread 2: [Wait] [Wait] [Lock] ← 1s wait
  Thread 3: [Wait] [Wait] [Wait] [Lock] ← 2s wait
  Thread 4: [Wait] [Wait] [Wait] [Wait] [Lock] ← 3s wait
  
  Total system time: 10s
  Instead of: 4s (all parallel)
```

### 7.2 Lock Bottleneck
```
Global synchronized object:

synchronized(globalLock) {
    // Everyone waits for this
}

Problem:
  - Thread pool threads accumulate
  - Waiting threads consume resources
  - System appears hung
  - Lock holder under pressure (might be slow)
  - Deadlock risk

Solution: 
  - Reduce lock scope
  - Use finer-grained locks
  - Use ConcurrentHashMap instead of synchronizedMap
```

### 7.3 Deadlock
```
Scenario:

Thread 1:
  synchronized(lockA) {
    synchronized(lockB) {
      // code
    }
  }

Thread 2:
  synchronized(lockB) {
    synchronized(lockA) {  // Waiting for lockA!
      // code
    }
  }

Timeline:
  Thread 1: [acquires lockA]
  Thread 2: [acquires lockB]
  Thread 1: [waits for lockB] ← held by Thread 2
  Thread 2: [waits for lockA] ← held by Thread 1
  
  DEADLOCK: Both wait forever
```

**Prevention:**
- Always acquire locks in same order
- Avoid nested locks
- Use timeouts

---

## SECTION 8: VOLATILE

### 8.1 When to Use volatile
```
Single variable visibility needed (not compound operations)

volatile int shutdown = 0;

Thread 1:
  shutdown = 1;  // Write visible immediately

Thread 2:
  while(shutdown == 0) {  // Always sees latest value
    doWork();
  }
```

### 8.2 volatile vs synchronized
```
volatile:
  ✓ Lighter weight than synchronized
  ✓ Guarantees visibility
  ✗ No atomicity
  ✗ No mutual exclusion
  Usage: Single variable visibility

synchronized:
  ✓ Guarantees atomicity
  ✓ Guarantees visibility
  ✓ Guarantees mutual exclusion
  ✗ Heavier weight
  Usage: Compound operations
```

---

## SECTION 9: PRODUCER-CONSUMER PATTERN

### 9.1 The Pattern
```
Scenario: Thread pool + Work queue

Main Thread:
  Task task = new Task();
  queue.put(task);  // Blocks if queue full

Worker Thread:
  Task task = queue.take();  // Blocks if queue empty
  task.execute();
```

### 9.2 BlockingQueue
```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100);

// Producer
void addTask(Task task) {
    try {
        queue.put(task);  // Blocks if full
    } catch (InterruptedException e) {
        // interrupted
    }
}

// Consumer (Worker)
void processQueue() {
    while(true) {
        try {
            Task task = queue.take();  // Blocks if empty
            task.execute();
        } catch (InterruptedException e) {
            break;
        }
    }
}
```

### 9.3 ThreadPoolExecutor
```
Actual implementation:

Tasks submitted
  ↓
  Core threads busy?
  ├─ NO: Create new thread immediately
  ├─ YES: Add to queue
  
Queue full?
  ├─ NO: Wait in queue
  ├─ YES: Create more threads (up to max)
  
Max threads reached?
  ├─ Queue still full?
  └─ YES: Reject task (throw RejectedExecutionException)
```

---

## SECTION 10: VOLATILE & MEMORY BARRIERS

### 10.1 Happens-Before Relationship
```
synchronized creates happens-before:

Lock acquisition:
  All subsequent operations see writes before release
  
volatile write:
  All subsequent reads see this write
  
Normal operation:
  Compiler can reorder (optimization)
```

### 10.2 Memory Barrier
```
volatile write:
  [Store barrier]
  Write forced to main memory
  
volatile read:
  [Load barrier]
  Fetch from main memory
  
Ensures: visibility across threads
Cost: Cannot cache in registers
```

---

## SECTION 11: COMMON PATTERNS

### 11.1 Double-Checked Locking (Singleton)
```java
public class Singleton {
    private static volatile Singleton instance;
    
    public static Singleton getInstance() {
        if(instance == null) {  // First check (no lock)
            synchronized(Singleton.class) {
                if(instance == null) {  // Second check (locked)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

Why volatile?
  - Ensures partially-constructed instance not seen
  - Memory barrier on constructor completion
```

### 11.2 Immutable Object
```java
public final class ImmutableUser {
    private final String name;
    private final int age;
    
    public ImmutableUser(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    // Getters only, no setters
}

// Thread-safe without synchronization
// Multiple threads can read same instance
// No mutations possible
```

### 11.3 Thread Confinement
```
Strategy: Each thread owns its data

Example: ThreadLocal

Thread 1: owns Connection 1
Thread 2: owns Connection 2
Thread 3: owns Connection 3

No sharing = no synchronization needed
```

---

## SECTION 12: PRODUCTION CONCURRENCY ISSUES

### 12.1 Thread Pool Exhaustion
```
Symptoms:
  - Sudden spike in latency
  - Queue depth high
  - Throughput drops
  - New requests rejected

Root cause:
  - Task takes too long
  - Blocking I/O not handled
  - Downstream service slow

Example:
  Threads: 10
  Task duration: 100ms → 1s (suddenly slow)
  
  Queue fills
  New requests wait 10+ seconds
  Client times out
  Retry storm begins
  Complete failure
```

### 12.2 Deadlock
```
Production deadlock scenario:

Thread 1 (request):
  synchronized(account1) {
    transfer to account2
    synchronized(account2) {
      // code
    }
  }

Thread 2 (request):
  synchronized(account2) {
    transfer to account1
    synchronized(account1) {  // BLOCKED
      // code
    }
  }

Result: 
  Both threads blocked
  Thread pool threads accumulate
  Eventually pool exhausted
  System hangs
```

### 12.3 Lock Contention Cascade
```
Scenario: Slow operation under synchronized

synchronized(globalState) {
    String result = slowDatabaseCall();  // Takes 5s
    globalState.update(result);
}

Effect:
  Thread 1: Holds lock 5s
  Thread 2: Waits 5s
  Thread 3: Waits 10s
  Thread 4: Waits 15s
  ...
  
  Load increase 100%
  Thread pool exhausted
  System collapse
```

### 12.4 Memory Leak in Concurrent Code
```
Scenario: ThreadLocal not cleaned

ThreadLocal<List> buffer = new ThreadLocal<>();

Thread pool worker:
  1. Get request
  2. buffer.set(new ArrayList<>())
  3. Process (forgot remove()!)
  4. Return to pool
  5. Next request reuses thread
  6. buffer.get() returns OLD list ← LEAK
  7. Memory accumulates
  8. Eventually OOM
```

---

## SECTION 13: JAVA MEMORY MODEL

### 13.1 Visibility Guarantees
```
synchronized:
  Entry: Load barrier (read latest)
  Exit: Store barrier (write out)

volatile:
  Read: Load barrier
  Write: Store barrier

Non-volatile, non-synchronized:
  No guarantees
  Compiler can cache in registers
  Other threads never see updates
```

### 13.2 Atomicity
```
Not atomic:
  count++  // Read, increment, write (3 steps)
  
Atomic:
  count.incrementAndGet()  // Atomic operation
  
synchronized(counter) {
    count++;  // Atomic within block
}
```

---

## SECTION 14: DEBUGGING CONCURRENCY ISSUES

### 14.1 Thread Dump Analysis
```
Run: jstack <pid> > threadump.txt

Look for:
  1. BLOCKED threads (waiting for lock)
  2. WAITING threads (on condition/lock)
  3. Threads holding locks
  4. Deadlock patterns
```

Example:
```
Thread-1: state=BLOCKED, lock=<0x123>
  at Object.wait()
  at ConcurrentHashMap.get()  ← waiting here

Thread-2: state=RUNNABLE, lock=<0x123>
  at MyService.update()
  ← Thread-2 holds the lock!
```

### 14.2 Heap Dump Analysis
```
Check for:
  - ThreadLocal not cleaned (retained objects)
  - Connections not returned to pool
  - Static collections growing
  
Use: jmap -dump:file=heap.hprof <pid>
Then: jhat heap.hprof
```

### 14.3 Profiling Contention
```
Use JProfiler or YourKit:
  - Lock contention times
  - Which threads wait where
  - How long they wait
  
Example output:
  Lock: global_state
  Wait time: 50%
  Waiting threads: 8
  
Solution: Split into finer-grained locks
```

---

## SECTION 15: BEST PRACTICES

### 15.1 General Principles
```
1. Minimize Shared Mutable State
   - Use immutables
   - Use ThreadLocal
   - Use thread confinement

2. Minimize Lock Contention
   - Reduce critical section
   - Use ConcurrentHashMap instead of synchronizedMap
   - Use lock striping
   - Use atomic operations

3. Avoid Nested Locks
   - Increases deadlock risk
   - Reduces parallelism
   - Hard to reason about

4. Always Clean ThreadLocal
   - try/finally with remove()
   - Use removeIfPresent()
```

### 15.2 Design Patterns
```
Immutable Objects
  ✓ No synchronization needed
  ✓ Safe for concurrent access
  ✓ Easier to reason about

Thread Confinement
  ✓ Each thread owns data
  ✓ No sharing = no sync needed
  ✓ ThreadLocal good fit

Lock Striping
  ✓ Multiple locks instead of one
  ✓ Different threads different locks
  ✓ ConcurrentHashMap example
```

---

## SECTION 16: INTERVIEW QUESTIONS

### 16.1 Common Questions
```
Q: What's the difference between synchronized and volatile?
A: 
  - synchronized: mutual exclusion + atomicity + visibility
  - volatile: visibility only (no atomicity)

Q: Can two threads execute synchronized methods simultaneously?
A: 
  - If same object: NO (same lock)
  - If different objects: YES (different locks)

Q: What causes deadlock?
A:
  - Circular lock dependency
  - Thread A holds lock X, needs lock Y
  - Thread B holds lock Y, needs lock X

Q: What's difference between Runnable and Callable?
A:
  - Runnable: void run() (no return value)
  - Callable: V call() (returns value, throws checked exception)

Q: How to create thread-safe singleton?
A:
  - Eager initialization
  - Double-checked locking with volatile
  - Bill Pugh singleton (nested class)
```

---

## KEY TAKEAWAYS

1. **Multiple threads + shared mutable state = synchronization required**
2. **synchronized = mutual exclusion + visibility guarantee**
3. **volatile = visibility only (not atomic)**
4. **ConcurrentHashMap > synchronizedMap for most cases**
5. **ThreadLocal requires cleanup to avoid leaks**
6. **Nested locks create deadlock risk**
7. **Lock contention is production killer (check with profiler)**
8. **Immutable objects are naturally thread-safe**
9. **Thread pool exhaustion cascades through system**
10. **Always acquire locks in consistent order**

---

*Use this KB as reference for writing thread-safe code, debugging production issues, and interview preparation.*
