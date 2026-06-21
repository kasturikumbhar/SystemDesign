Perfect. For these final two, the best format is Notion-style documentation with colored callout boxes, layered diagrams, and visual flows (same style as the previous ones).

⸻

📗 KB_005: Java Internals & JVM Deep Dive

⸻

☕ JAVA EXECUTION ARCHITECTURE

⸻

🟦 Java Compilation Pipeline

                Source Code (.java)
                         │
                         ▼
                 javac Compiler
                         │
                         ▼
                  Bytecode (.class)
                         │
                         ▼
                     JVM Runtime
                         │
       ┌─────────────────┼──────────────────┐
       ▼                 ▼                  ▼
 Class Loader      Runtime Memory      Execution Engine
                                             │
                                  ┌──────────┴──────────┐
                                  ▼                     ▼
                              Interpreter             JIT Compiler
                                  │                     │
                                  └────────► Native Machine Code

⸻

🟩 Why JVM Exists

Write Once
      │
      ▼
Compile to Bytecode
      │
      ▼
Run Anywhere
┌──────────────┬──────────────┬──────────────┐
Windows       Linux          MacOS

⸻

PART 1 — JVM ARCHITECTURE

⸻

🟦 Runtime Memory Areas

                     JVM Memory
┌──────────────────────────────────────────┐
│ Heap (Shared)                            │
│                                          │
│  Young Gen                               │
│    Eden                                  │
│    S0                                    │
│    S1                                    │
│                                          │
│  Old Generation                          │
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│ Metaspace                                │
│ Class metadata                           │
└──────────────────────────────────────────┘
Per Thread
──────────────────────────────────
Stack
PC Register
Native Method Stack

⸻

🟨 Stack

Each thread owns a stack.

main()
 │
 ▼
service()
 │
 ▼
calculate()
Top of Stack

Stores:

* Local variables
* Method calls
* References

⸻

🟩 Heap

Stores objects.

new User()
new ArrayList()
↓
Heap

Shared by all threads.

⸻

PART 2 — CLASS LOADING

⸻

ClassLoader Hierarchy

Bootstrap Loader
        │
        ▼
Extension Loader
        │
        ▼
Application Loader
        │
        ▼
Custom ClassLoader

⸻

Parent Delegation

Load String.class
App Loader
     │
Parent?
     ▼
Ext Loader
     │
Parent?
     ▼
Bootstrap
     │
Found

Prevents duplicate classes.

⸻

PART 3 — EXECUTION ENGINE

⸻

Interpreter vs JIT

Interpreter

Bytecode
Instruction 1
Instruction 2
Instruction 3
Execute one-by-one

Simple but slow.

⸻

JIT Compiler

Frequently executed code:

Method hot?
YES
 │
 ▼
Compile to native machine code
 │
 ▼
Cache

Much faster.

⸻

HotSpot Optimization

Inlining
Loop Unrolling
Escape Analysis
Dead Code Elimination

⸻

PART 4 — HEAP ORGANIZATION

⸻

                Heap
┌────────────────────────────────┐
│ Young Generation               │
│                                │
│ Eden     S0     S1             │
└────────────────────────────────┘
┌────────────────────────────────┐
│ Old Generation                 │
└────────────────────────────────┘

⸻

Object Lifecycle

new Object()
↓
Eden
↓
Minor GC
↓
Survivor S0
↓
Survivor S1
↓
Old Generation
↓
Major GC

⸻

PART 5 — GARBAGE COLLECTION

⸻

Serial GC

Application STOP
GC Thread
Application RESUME

Simple but pauses entire app.

⸻

Parallel GC

CPU1 → GC
CPU2 → GC
CPU3 → GC
CPU4 → GC

High throughput.

⸻

G1GC

(Default modern JVM)

Heap divided into Regions
□□□□□□□□□
□□□□□□□□□
□□□□□□□□□
Collect only regions with most garbage

Advantages:

✅ Lower pauses

✅ Large heaps

⸻

ZGC

Application Running
GC Running Simultaneously
No stop-the-world pause

Pause:

< 1ms

Suitable for:

* Low latency systems
* Huge heaps

⸻

PART 6 — JAVA MEMORY MODEL

⸻

Visibility Problem

Thread A:
flag = true
Thread B:
while(flag==false)

Without synchronization:

CPU Cache A
flag=true
CPU Cache B
flag=false

Thread B may never see update.

⸻

volatile

volatile boolean flag;

Guarantees:

✅ Visibility

❌ Atomicity

⸻

Happens-Before

Thread A
write x=10
unlock()
       Happens Before
Thread B
lock()
read x=10

Guarantees memory visibility.

⸻

PART 7 — SYNCHRONIZATION

⸻

synchronized

Thread 1
LOCK
execute
UNLOCK
Thread 2 waits

Mutual exclusion.

⸻

ReentrantLock

More features:

tryLock()
timeout
fairness
condition variables

⸻

CAS (Compare-And-Swap)

Used by:

AtomicInteger
ConcurrentHashMap
Read value
Still same?
YES → update
NO → retry

Lock-free programming.

⸻

PART 8 — JAVA FEATURES

⸻

Java 8

Lambda

list.stream()
.filter()
.map()
.collect()

⸻

Stream API

Collection
↓
Filter
↓
Map
↓
Reduce

⸻

Java 14+

Records

record User(
 String name,
 int age
)

Immutable DTO.

⸻

Java 17

Sealed Classes

Shape
├── Circle
├── Square
└── Triangle

Restricts inheritance.

⸻

Java 21

Virtual Threads

Traditional:

1 Thread = 1 OS Thread

Virtual Threads:

100000 Virtual Threads
↓
Few Carrier Threads

Huge scalability improvement.

⸻

PART 9 — REFLECTION

Class object
↓
Inspect fields
↓
Inspect methods
↓
Invoke dynamically

Used by:

* Spring
* Hibernate
* Jackson

⸻

PART 10 — MONITORING

⸻

jstack

Thread dumps

jstack PID

Find deadlocks.

⸻

jmap

Heap dump

jmap -dump

Analyze memory leaks.

⸻

jstat

GC statistics

jstat -gc PID

⸻

VisualVM

CPU
Memory
Threads
Heap
GC

⸻

JVM TUNING

-Xms4g
-Xmx4g
-XX:+UseG1GC

Monitor:

GC pause time
Heap occupancy
Allocation rate

⸻

🧠 Mental Model

Java Source
      │
      ▼
Bytecode
      │
      ▼
Class Loader
      │
      ▼
Execution Engine
      │
      ▼
Heap + Stack
      │
      ▼
Garbage Collector
      │
      ▼
Monitoring + Tuning

⸻

