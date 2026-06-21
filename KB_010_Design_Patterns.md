🎨 Knowledge Base: Design Patterns

Creational, Structural & Behavioral Patterns with Real-World Applications

⸻

🏗 PART 1 — CREATIONAL PATTERNS

Purpose: Control how objects are created

⸻

1. Singleton Pattern

🟦 Goal

Ensure only one object instance exists and provide a global access point.

⸻

❌ Without Singleton

new DatabaseConnection()
↓
Connection #1
new DatabaseConnection()
↓
Connection #2
new DatabaseConnection()
↓
Connection #3

Problems

🔴 Too many DB connections

🔴 Wasted memory

🔴 Resource exhaustion

⸻

✅ With Singleton

          getInstance()
                ↓
        ┌─────────────────┐
        │ DatabaseConnection│
        │   Single Object  │
        └─────────────────┘
           ↑           ↑
         db1         db2
        db1 == db2

⸻

Implementation

class DatabaseConnection {
    private static DatabaseConnection instance;
    private DatabaseConnection() {}
    public static DatabaseConnection getInstance() {
        if(instance == null)
            instance = new DatabaseConnection();
        return instance;
    }
}

⸻

Thread Safe Singleton

Thread 1
    ↓
instance == null ?
    ↓ yes
LOCK
    ↓
create object
Thread 2
    ↓
waits for lock

Double-check locking:

if(instance == null){
   synchronized(...){
      if(instance == null){
         instance = new DatabaseConnection();
      }
   }
}

⸻

Better: Eager Initialization

private static final DatabaseConnection INSTANCE
        = new DatabaseConnection();

Created at class loading → automatically thread-safe.

⸻

✅ Use Cases

🟢 Logger

🟢 Configuration Manager

🟢 Database Pool

🟢 Thread Pool

⸻

⚠ Downsides

🔴 Global state

🔴 Harder unit testing

🔴 Hidden dependencies

⸻

2. Builder Pattern

🟦 Goal:

Build complex objects step-by-step.

⸻

❌ Constructor Hell

new User(
 name,
 email,
 phone,
 address,
 city,
 state,
 zip,
 age,
 company,
 job
);

Problems

🔴 Order difficult to remember

🔴 Null parameters

🔴 Hard to extend

⸻

✅ Builder

UserBuilder
     ↓
setName()
     ↓
setEmail()
     ↓
setPhone()
     ↓
build()
     ↓
User object

⸻

Example

User user = new UserBuilder()
                .setName("John")
                .setEmail("john@ex.com")
                .setPhone("123")
                .build();

⸻

Visual

             UserBuilder
                   |
      -------------------------
      |           |           |
   name        email       phone
      |           |           |
      ------------- ----------
                    |
                 build()
                    |
                  User

⸻

Benefits

🟢 Readable

🟢 Optional fields

🟢 Easy extension

🟢 Immutable objects

⸻

Use Cases

✅ DTOs

✅ Config objects

✅ APIs

✅ Lombok @Builder

⸻

3. Factory Pattern

🟦 Goal:

Hide object creation logic.

⸻

❌ Without Factory

if CreditCard
    new CreditCardProcessor()
if PayPal
    new PayPalProcessor()
if Stripe
    new StripeProcessor()

Problems:

🔴 Tight coupling

🔴 Huge if-else chains

⸻

✅ Factory

             Payment Type
                    |
                    ▼
         PaymentProcessorFactory
           /         |        \
          /          |         \
CreditCard       PayPal      Stripe
Processor       Processor    Processor

⸻

Code

PaymentProcessor processor =
      PaymentProcessorFactory.create(type);

⸻

Benefits

🟢 Loose coupling

🟢 Centralized creation

🟢 Easy to add implementations

⸻

Real Examples

✔ Spring Dependency Injection

✔ JDBC Drivers

✔ Plugin systems

⸻

🧱 PART 2 — STRUCTURAL PATTERNS

Focus on how classes are connected

⸻

4. Decorator Pattern

Add features dynamically without subclass explosion.

⸻

Problem

Imagine:

FileInputStream
BufferedInputStream
CompressedInputStream
EncryptedInputStream
BufferedCompressedInputStream
BufferedEncryptedInputStream
...

Explosion of subclasses.

⸻

Solution

Wrap objects:

FileInputStream
      ↑
BufferedInputStream
      ↑
CompressedInputStream
      ↑
EncryptedInputStream

Each layer adds behavior.

⸻

Example

new EncryptedInputStream(
     new CompressedInputStream(
         new BufferedInputStream(
             new FileInputStream()
         )
     )
);

⸻

Visual

Request
   ↓
Encryption
   ↓
Compression
   ↓
Buffering
   ↓
File

⸻

Real World

Java I/O

BufferedInputStream(
    FileInputStream()
)

Spring Security Filters

Request
 ↓
Authentication Filter
 ↓
CSRF Filter
 ↓
Session Filter
 ↓
Controller

⸻

5. Adapter Pattern

Makes incompatible interfaces work together.

⸻

Problem

Third-party expects:

DataInterface

But your object:

MyData

They’re incompatible.

⸻

Solution

MyData
   ↓
DataAdapter
   ↓
DataInterface
   ↓
Third-party API

⸻

Example

MyData data = new MyData();
DataInterface adapted =
       new DataAdapter(data);
process(adapted);

⸻

Real Examples

JDBC

MySQL Driver
Postgres Driver
Oracle Driver
       ↓
JDBC Adapter
       ↓
Connection Interface

⸻

Use Cases

✔ Legacy systems

✔ Third-party libraries

✔ Interface conversion

⸻

🔄 PART 3 — BEHAVIORAL PATTERNS

Focus on how objects communicate

⸻

6. Observer Pattern

One-to-many notifications.

⸻

Without Observer

OrderService
   |
   ├── Email Service
   ├── Inventory Service
   ├── Analytics Service
   └── SMS Service

Everything tightly coupled.

⸻

With Observer

          OrderService
                |
    ----------------------------
    |            |            |
 EmailObs    InventoryObs  AnalyticsObs

⸻

Event Flow

Create Order
      ↓
Save Order
      ↓
Notify Observers
      ↓
 ┌────────┬─────────┬────────┐
 ↓        ↓         ↓
Email   Inventory  Analytics

⸻

Benefits

🟢 Open/Closed Principle

🟢 Loose coupling

🟢 Easy extension

⸻

Modern Version

OrderCreated Event
          ↓
     Event Bus
 ┌────────┼────────┐
 ↓        ↓        ↓
Email   Inventory Analytics

Examples:

✔ Spring Events

✔ Kafka

✔ RabbitMQ

⸻

7. Strategy Pattern

Swap algorithms dynamically.

⸻

Problem

Huge if-else:

if PDF
if EXCEL
if JSON

⸻

Solution

          ReportGenerator
                  |
      ------------------------
      |           |          |
 PDF Strategy Excel Strategy JSON Strategy

⸻

Runtime Selection

generator =
      new ReportGenerator(
            new PdfStrategy()
      );

⸻

Visual

Generate Report
       |
  Choose Strategy
       |
---------------------
|        |          |
PDF     Excel      JSON

⸻

Real Examples

✔ Payment Methods

Credit Card
PayPal
Apple Pay
UPI

✔ Sorting Algorithms

QuickSort

MergeSort

HeapSort

⸻

8. Command Pattern

Encapsulate actions into objects.

⸻

Structure

Button Click
      ↓
 Command Object
      ↓
 execute()
      ↓
 Action Performed

⸻

Undo Support

Execute Command
       ↓
Push into History Stack
       ↓
Undo
       ↓
Pop Command
       ↓
undo()

⸻

Example

Rename old.txt → new.txt
        ↓
Stored in stack
Undo
        ↓
Rename new.txt → old.txt

⸻

Diagram

            UndoManager
                  |
         -----------------
         |               |
     execute()         undo()
         |               |
         ▼               ▲
      Command Stack (History)

⸻

Real World

Text Editors

Ctrl+Z
Ctrl+Y

⸻

Job Queues

Commands
     ↓
Queue
     ↓
Workers
     ↓
Execute

⸻

Database Transactions

Transaction Log
     ↓
Replay after crash

⸻

🧠 Pattern Categories Summary

Category	Pattern	Purpose
🏗 Creational	Singleton	Single instance
🏗 Creational	Builder	Complex object creation
🏗 Creational	Factory	Hide object creation
🧱 Structural	Decorator	Add behavior dynamically
🧱 Structural	Adapter	Interface conversion
🔄 Behavioral	Observer	Event notification
🔄 Behavioral	Strategy	Switch algorithms
🔄 Behavioral	Command	Encapsulate actions

⸻

🎯 Pattern Selection Cheat Sheet

Need ONE object?
        ↓
     Singleton
Too many constructor params?
        ↓
      Builder
Need object creation abstraction?
        ↓
      Factory
Need extra features?
        ↓
     Decorator
Need incompatible APIs to work?
        ↓
      Adapter
Need event notifications?
        ↓
      Observer
Need interchangeable algorithms?
        ↓
      Strategy
Need undo/redo or queues?
        ↓
      Command

⸻

⭐ Most Common in Real Systems

Spring Framework
├── Singleton
├── Factory
├── Strategy
├── Observer
└── Decorator
Kafka
├── Observer
├── Command
└── Strategy
Java Collections
└── Strategy
Java IO
└── Decorator
Spring MVC
└── Adapter

⸻

🚀 Key Takeaways

🟢 Patterns solve recurring problems.

🟢 Prefer composition over inheritance.

🟢 Avoid huge if-else chains.

🟢 Use patterns when complexity demands them.

🟢 Frameworks like Spring already implement many patterns.

⸻

Design Patterns are reusable blueprints—not rules. Use them to simplify code, not complicate it.
