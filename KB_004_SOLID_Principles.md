# Knowledge Base: SOLID Principles Deep Dive
## Object-Oriented Design for Scalable & Maintainable Systems

---

## INTRODUCTION

### Why SOLID Matters
```
Bad Design:
  - Hard to modify
  - Changes break other code
  - Tight coupling
  - Difficult to test
  - Expensive maintenance

SOLID Design:
  - Changes localized
  - Loose coupling
  - Easy to test
  - Cheap to maintain
  - Extensible systems
```

---

## PRINCIPLE 1: Single Responsibility Principle (SRP)

### Definition
**A class should have one, and only one, reason to change.**

### Problem It Solves
```java
// BAD: Multiple reasons to change
public class User {
    private String name;
    private String email;
    
    // Persistence logic
    public void save() {
        // Save to database
    }
    
    // Serialization logic
    public String toJSON() {
        // Convert to JSON
    }
    
    // Email validation logic
    public boolean isValidEmail() {
        // Validate email format
    }
    
    // Business logic
    public void sendWelcomeEmail() {
        // Send email
    }
}

Reasons to change:
  1. Change database storage format
  2. Change JSON serialization format
  3. Change email validation rules
  4. Change email sending service
  
User class has TOO MANY responsibilities!
```

### Solution
```java
// GOOD: Single responsibility each

// 1. Domain object (data only)
public class User {
    private String name;
    private String email;
    
    public String getName() { return name; }
    public String getEmail() { return email; }
}

// 2. Persistence
public class UserRepository {
    public void save(User user) {
        // Save to database
    }
}

// 3. Serialization
public class UserJSONSerializer {
    public String serialize(User user) {
        // Convert to JSON
    }
}

// 4. Email validation
public class EmailValidator {
    public boolean isValid(String email) {
        // Validate format
    }
}

// 5. Email service
public class EmailService {
    public void sendWelcomeEmail(User user) {
        // Send email
    }
}
```

### Benefits
```
Each class has single reason to change:

UserRepository changes:
  - Change database schema
  - Change ORM framework
  - ✗ Does NOT affect validation, serialization, email

EmailValidator changes:
  - Change email validation rules
  - ✗ Does NOT affect storage, serialization, email sending

Easy to test:
  - Test UserRepository in isolation
  - Test EmailValidator independently
  - Mock dependencies easily

Easy to reuse:
  - EmailValidator used in multiple places
  - UserRepository used consistently
  - Each can evolve independently
```

---

## PRINCIPLE 2: Open/Closed Principle (OCP)

### Definition
**Software entities should be open for extension, but closed for modification.**

### Problem It Solves
```java
// BAD: Must modify for every new payment method
public class PaymentProcessor {
    
    public void process(Payment payment) {
        if(payment.getType().equals("CREDIT_CARD")) {
            processCreditCard(payment);
        } else if(payment.getType().equals("DEBIT_CARD")) {
            processDebitCard(payment);
        } else if(payment.getType().equals("PAYPAL")) {
            processPayPal(payment);
        } else if(payment.getType().equals("BITCOIN")) {  // NEW!
            processBitcoin(payment);
        }
    }
    
    private void processCreditCard(Payment p) { /* */ }
    private void processDebitCard(Payment p) { /* */ }
    private void processPayPal(Payment p) { /* */ }
    private void processBitcoin(Payment p) { /* */ }  // NEW METHOD
}

Problem:
  - Every new payment type requires modifying PaymentProcessor
  - Risk of breaking existing functionality
  - Class gets bigger, more complex
  - Violates SRP (multiple reasons to change)
```

### Solution
```java
// GOOD: Closed for modification, open for extension

// Abstract interface (closed)
public interface PaymentGateway {
    void process(Payment payment);
    void refund(Payment payment);
}

// Implementations (open for extension)
public class CreditCardGateway implements PaymentGateway {
    @Override
    public void process(Payment payment) {
        // Credit card specific logic
    }
    
    @Override
    public void refund(Payment payment) {
        // Credit card specific refund
    }
}

public class DebitCardGateway implements PaymentGateway {
    @Override
    public void process(Payment payment) {
        // Debit card specific logic
    }
    
    @Override
    public void refund(Payment payment) {
        // Debit card specific refund
    }
}

public class PayPalGateway implements PaymentGateway {
    @Override
    public void process(Payment payment) {
        // PayPal specific logic
    }
    
    @Override
    public void refund(Payment payment) {
        // PayPal specific refund
    }
}

// NEW payment method: Just add new class
public class BitcoinGateway implements PaymentGateway {
    @Override
    public void process(Payment payment) {
        // Bitcoin specific logic
    }
    
    @Override
    public void refund(Payment payment) {
        // Bitcoin specific refund
    }
}

// Processor (CLOSED for modification)
@Service
public class PaymentProcessor {
    @Autowired
    private Map<String, PaymentGateway> gateways;
    
    public void process(Payment payment) {
        PaymentGateway gateway = gateways.get(payment.getType());
        gateway.process(payment);
    }
    
    public void refund(Payment payment) {
        PaymentGateway gateway = gateways.get(payment.getType());
        gateway.refund(payment);
    }
}

// Configuration
@Configuration
public class PaymentConfig {
    @Bean("CREDIT_CARD")
    public PaymentGateway creditCardGateway() {
        return new CreditCardGateway();
    }
    
    @Bean("DEBIT_CARD")
    public PaymentGateway debitCardGateway() {
        return new DebitCardGateway();
    }
    
    @Bean("PAYPAL")
    public PaymentGateway payPalGateway() {
        return new PayPalGateway();
    }
    
    @Bean("BITCOIN")
    public PaymentGateway bitcoinGateway() {
        return new BitcoinGateway();  // NEW: just add here
    }
}
```

### Benefits
```
Adding Bitcoin payment:
  1. Create BitcoinGateway class (NEW)
  2. Register in @Configuration (MODIFY only config)
  3. ✓ PaymentProcessor UNCHANGED
  4. ✓ No risk of breaking existing
  5. ✓ Easy to test new gateway in isolation

Pattern: Strategy Pattern
  - Closed: PaymentProcessor
  - Open: New PaymentGateway implementations
```

---

## PRINCIPLE 3: Liskov Substitution Principle (LSP)

### Definition
**Objects in a program should be replaceable with instances of their subtypes without breaking the application.**

### Problem It Solves
```java
// BAD: Subtypes break parent contract

public class Bird {
    public void fly() {
        // fly for 5 minutes
    }
}

public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Penguins can't fly!");
    }
}

// Client code
public void makeBirdFly(Bird bird) {
    bird.fly();  // Assumes all birds can fly
}

// Using it
Bird bird = new Penguin();
makeBirdFly(bird);  // CRASH! Throws exception

Problem:
  - Penguin violates Bird contract
  - Cannot substitute Penguin for Bird
  - Client code breaks
  - Violates type safety promise
```

### Solution
```java
// GOOD: Correct hierarchy

public abstract class Bird {
    // Empty or provide default
}

public class FlyingBird extends Bird {
    public void fly() {
        // Can fly for 5 minutes
    }
}

public class Penguin extends Bird {
    public void swim() {
        // Can swim instead
    }
}

// Client code
public void makeBirdFly(FlyingBird bird) {
    bird.fly();  // Contract guaranteed
}

public void makeBirdSwim(Penguin penguin) {
    penguin.swim();
}

// Using it
Bird bird = new Penguin();
if(bird instanceof FlyingBird) {
    makeBirdFly((FlyingBird) bird);
} else {
    makeBirdSwim((Penguin) bird);
}
```

---

## PRINCIPLE 4: Interface Segregation Principle (ISP)

### Definition
**Many client-specific interfaces are better than one general-purpose interface.**

### Problem It Solves
```java
// BAD: Fat interface

public interface Worker {
    void work();
    void eat();
    void sleep();
}

public class Human implements Worker {
    @Override
    public void work() {
        // work implementation
    }
    
    @Override
    public void eat() {
        // eat implementation
    }
    
    @Override
    public void sleep() {
        // sleep implementation
    }
}

public class Robot implements Worker {
    @Override
    public void work() {
        // work implementation
    }
    
    @Override
    public void eat() {
        throw new UnsupportedOperationException("Robots don't eat!");
    }
    
    @Override
    public void sleep() {
        throw new UnsupportedOperationException("Robots don't sleep!");
    }
}
```

### Solution
```java
// GOOD: Segregated interfaces

public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public class Human implements Workable, Eatable, Sleepable {
    @Override
    public void work() { /* */ }
    
    @Override
    public void eat() { /* */ }
    
    @Override
    public void sleep() { /* */ }
}

public class Robot implements Workable {
    @Override
    public void work() { /* */ }
    // No forced eat() or sleep()!
}
```

---

## PRINCIPLE 5: Dependency Inversion Principle (DIP)

### Definition
**High-level modules should not depend on low-level modules. Both should depend on abstractions.**

### Problem It Solves
```java
// BAD: High-level depends on low-level

public class UserService {
    private MySQLDatabase db;  // Concrete dependency
    
    public UserService() {
        this.db = new MySQLDatabase();  // Tight coupling
    }
    
    public User getUserById(Long id) {
        return db.query("SELECT * FROM users WHERE id = " + id);
    }
}

Problems:
  - UserService tightly coupled to MySQL
  - Cannot use PostgreSQL without modifying UserService
  - Cannot mock for testing
  - High-level logic depends on low-level database
  - Violates OCP (need to modify to change database)
  - Cannot swap implementations
```

### Solution
```java
// GOOD: Depend on abstractions

// Abstraction
public interface UserRepository {
    User getUserById(Long id);
}

// Low-level implementations
public class MySQLUserRepository implements UserRepository {
    @Override
    public User getUserById(Long id) {
        // MySQL specific query
    }
}

public class PostgreSQLUserRepository implements UserRepository {
    @Override
    public User getUserById(Long id) {
        // PostgreSQL specific query
    }
}

// High-level service
@Service
public class UserService {
    private final UserRepository repository;  // Abstraction
    
    @Autowired
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
    
    public User getUserById(Long id) {
        return repository.getUserById(id);
    }
}

// Configuration
@Configuration
public class RepositoryConfig {
    @Bean
    public UserRepository userRepository() {
        // Can switch implementations here
        return new PostgreSQLUserRepository();
    }
}
```

---

## INTEGRATION EXAMPLE

See GitHub KB for complete Notification System example showing all 5 SOLID principles together.

---

## KEY TAKEAWAYS

1. **SRP**: One reason to change per class
2. **OCP**: Extend via inheritance/interfaces, not modification
3. **LSP**: Subtypes must honor parent contract
4. **ISP**: Segregate fat interfaces into smaller ones
5. **DIP**: Depend on abstractions, not concretions
6. **Together**: SOLID creates maintainable, testable, flexible systems
7. **Practice**: Takes time to internalize (design experience matters)
8. **Tradeoff**: SOLID code might be slightly more complex initially
9. **Value**: Pays off in production (easier changes, fewer bugs)
10. **Interview**: Understand patterns and when to apply them

---

*Master SOLID to design systems that scale in complexity, not just traffic.*
