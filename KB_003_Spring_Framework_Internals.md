# Knowledge Base: Spring Framework Internals
## Deep Dive into Spring Core, AOP, Security, WebFlux & Lifecycle

---

## PART 1: SPRING FUNDAMENTALS

### 1.1 What is Spring?
```
Spring = Dependency Injection Framework + Ecosystem

Core:
  - IoC Container (ApplicationContext)
  - Dependency Injection
  - AOP (Aspect Oriented Programming)
  
Extensions:
  - Spring MVC (web)
  - Spring Data (persistence)
  - Spring Security (authentication)
  - Spring Cloud (distributed systems)
  - Spring WebFlux (reactive)
```

### 1.2 ApplicationContext
```
ApplicationContext = Container that manages beans

Initialization:
  new ClassPathXmlApplicationContext("beans.xml")
  or
  new AnnotationConfigApplicationContext(AppConfig.class)
  or (Spring Boot)
  SpringApplication.run(MyApplication.class)

Responsibilities:
  - Load bean definitions
  - Instantiate beans
  - Resolve dependencies
  - Handle lifecycle
  - Create proxies for AOP
  - Manage singletons
```

---

## PART 2: DEPENDENCY INJECTION

### 2.1 Injection Methods
```java
// 1. Constructor Injection (Preferred)
@Component
public class UserService {
    private final UserRepository repo;
    
    public UserService(UserRepository repo) {
        this.repo = repo;  // Dependencies required
    }
}

// 2. Setter Injection
@Component
public class UserService {
    private UserRepository repo;
    
    @Autowired
    public void setRepository(UserRepository repo) {
        this.repo = repo;  // Dependencies optional
    }
}

// 3. Field Injection (NOT recommended)
@Component
public class UserService {
    @Autowired
    private UserRepository repo;  // Harder to test, less flexible
}
```

**Constructor injection advantages:**
- Dependencies explicit
- Immutability possible
- Testing easier (no reflection needed)
- Failure at startup (not runtime)

### 2.2 Bean Lifecycle
```
1. Bean Definition Registration
   @Component, @Service, @Repository annotations
   or programmatic registration

2. Bean Instantiation
   Spring calls constructor

3. Dependency Injection
   Set fields / call setters / call methods
   Autowired dependencies resolved

4. BeanPostProcessor.postProcessBeforeInitialization()
   Custom logic before initialization

5. InitializingBean.afterPropertiesSet() OR @PostConstruct
   Bean can initialize itself

6. BeanPostProcessor.postProcessAfterInitialization()
   Custom logic after initialization
   **AOP proxies created here**

7. Bean Ready for Use
   ApplicationContext gives to requesting code

8. Application Shutdown
   Spring closes context

9. DisposableBean.destroy() OR @PreDestroy
   Cleanup resources
```

### 2.3 Bean Scopes
```
@Scope("singleton") - DEFAULT
  ✓ One instance per ApplicationContext
  ✓ Shared across all requests
  ⚠️ Must be thread-safe
  
  Production use:
    - Services
    - Repositories
    - Utilities
    
  NOT thread-safe example:
    @Service
    public class BadService {
        private User currentUser;  // DANGER: shared across threads!
    }

@Scope("prototype")
  ✓ New instance every time
  ✓ No sharing needed
  ✓ Not thread-safe concerns
  
  Production use:
    - Request-handling objects
    - Data containers

@Scope("request") - Web only
  ✓ One instance per HTTP request
  ✓ Same request sees same bean
  ✓ Different requests see different beans
  
  Production use:
    - Request-scoped dependencies
    - Request context
    
  Implementation:
    RequestContextHolder.getRequestAttributes()

@Scope("session") - Web only
  ✓ One instance per user session
  ✓ Same user always sees same bean
  
  Production use:
    - User state
    - Shopping cart
```

---

## PART 3: ASPECT ORIENTED PROGRAMMING (AOP)

### 3.1 Problem AOP Solves
```
Without AOP:

@Service
public class UserService {
    public void createUser(User user) {
        // LOGGING
        logger.info("Creating user: " + user.getName());
        
        // VALIDATION
        if(user.getAge() < 18) {
            throw new ValidationException(...);
        }
        
        // BUSINESS LOGIC
        user = userRepository.save(user);
        
        // METRICS
        metrics.recordCreate(System.currentTimeMillis());
        
        // CACHING
        cache.put(user.getId(), user);
        
        return user;
    }
}

Problems:
  - Cross-cutting concerns mixed with business logic
  - Duplication across multiple methods
  - Hard to modify (changes needed everywhere)
  - Business logic buried under infrastructure
```

### 3.2 AOP with Spring
```java
// Aspect: Interceptor for cross-cutting concerns
@Aspect
@Component
public class LoggingAspect {
    
    @Around("@annotation(Timed)")
    public Object measureTime(ProceedingJoinPoint pjp) 
            throws Throwable {
        long start = System.currentTimeMillis();
        
        try {
            return pjp.proceed();  // Execute actual method
        } finally {
            long duration = System.currentTimeMillis() - start;
            logger.info("Method took: " + duration + "ms");
        }
    }
}

// Usage:
@Service
public class UserService {
    @Timed
    public User createUser(User user) {
        // Just business logic
        return userRepository.save(user);
    }
}
```

### 3.3 AOP Internals
```
Spring AOP uses Proxies:

1. During Bean Initialization:
   Spring detects @Aspect
   Scans for @Around, @Before, @After
   
2. For each matching method:
   Creates proxy wrapping original

3. At runtime:
   Call goes to proxy
   Proxy → Interceptor (aspect)
   Interceptor → Original method
   Interceptor → Back

Flow:
  Caller
    ↓
  Proxy (created by Spring)
    ↓
  Aspect (interceptor)
    ↓
  Original Method
    ↓
  Aspect (cleanup)
    ↓
  Proxy
    ↓
  Caller (with result)
```

### 3.4 Advice Types
```
@Before: Execute before method
  public void logBefore(JoinPoint jp) {
    logger.info("Calling: " + jp.getSignature());
  }

@After: Execute after method (always)
  public void cleanup(JoinPoint jp) {
    logger.info("Method finished");
  }

@AfterReturning: Execute after successful return
  public void logReturn(JoinPoint jp, Object result) {
    logger.info("Result: " + result);
  }

@AfterThrowing: Execute if exception thrown
  public void logException(JoinPoint jp, Exception ex) {
    logger.error("Exception: " + ex);
  }

@Around: Execute before AND after (most powerful)
  public Object aroundAdvice(ProceedingJoinPoint pjp) 
      throws Throwable {
    // Before
    try {
        return pjp.proceed();  // Execute actual method
    } finally {
        // After (cleanup)
    }
  }
```

### 3.5 Pointcut Expressions
```
@Around("execution(public * com.example.service.*Service.*(...))")
  Match all public methods in *Service classes in com.example.service

@Around("@annotation(Logged)")
  Match all methods with @Logged annotation

@Around("bean(userService)")
  Match all methods in bean named "userService"

@Around("within(com.example..*)")
  Match all methods in com.example package and subpackages

Combinations:
  @Around("execution(...) && args(id, name)")
  Match execution AND has parameters id and name
```

---

## PART 4: SPRING SECURITY

### 4.1 Authentication
```
Problem: Verifying who user is

Without Spring Security:
  @PostMapping("/login")
  public String login(String username, String password) {
      User user = userRepository.findByUsername(username);
      if(user != null && user.getPassword().equals(password)) {
          // Set session attribute
          httpSession.setAttribute("user", user);
          return "login_success";
      }
      return "login_fail";
  }

Problems:
  - Password stored plaintext (TERRIBLE!)
  - Session management manual
  - CSRF protection missing
  - Password comparison timing attack vulnerability
  - No centralized control

With Spring Security:
  - Automatic password encoding (BCrypt)
  - Session management automatic
  - CSRF protection built-in
  - Multiple authentication providers
  - Centralized configuration
```

### 4.2 Authentication Internals
```
Request arrives:
  ↓
SecurityFilterChain intercepts
  ↓
AuthenticationFilter extracts credentials
  (username/password from request)
  ↓
AuthenticationManager processes
  (delegates to AuthenticationProvider)
  ↓
AuthenticationProvider verifies
  ✓ User exists in repository
  ✓ Password matches (using PasswordEncoder)
  ✓ Account not locked
  ✓ Account not expired
  ↓
Returns Authentication object
  (successful or throws exception)
  ↓
SecurityContext stores Authentication
  (in ThreadLocal + Session)
  ↓
Request continues
  (@PreAuthorize checks can happen)
```

### 4.3 Authorization
```
Scenario: Restrict who can access what

With Spring Security:

@RestController
@RequestMapping("/api")
public class AdminController {
    
    @DeleteMapping("/users/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
    
    @GetMapping("/profile")
    @PreAuthorize("authentication.name == #username")
    public User getProfile(@PathVariable String username) {
        return userService.findByUsername(username);
    }
}

Checking authorization:
  1. AuthenticationContext retrieved from ThreadLocal
  2. Roles/authorities checked
  3. Expression evaluated
  4. ✓ Allowed → method executes
  5. ✗ Denied → AccessDeniedException thrown
```

### 4.4 Security Configuration
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/public/**").permitAll()
                .antMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            .and()
                .formLogin()
                    .loginPage("/login")
                .and()
                    .logout()
                    .logoutSuccessUrl("/");
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();  // Salt + hash
    }
}

URL Security:
  /public/** → Anyone
  /admin/** → ADMIN role only
  /** → Must be authenticated
```

---

## PART 5: SPRING WEBFLUX (REACTIVE)

### 5.1 Problem Spring WebFlux Solves
```
Traditional Spring MVC:
  Request
    ↓
  Thread assigned (from pool)
  Thread blocked waiting for DB
  Thread blocked waiting for API
  Thread blocked serializing JSON
    ↓
  Response

Problems:
  - Thread tied up while blocked
  - Thread pool limited (100-200 threads)
  - High concurrency needs hundreds of threads
  - Context switching overhead

Spring WebFlux:
  Request
    ↓
  Event loop thread takes request
    ↓
  Schedules async work (DB, API)
    ↓
  Thread released for other requests
    ↓
  When result available
    ↓
  Thread resumes pipeline
    ↓
  Response

Advantages:
  - Few threads handle many requests
  - Efficient utilization of resources
  - Better scalability
```

### 5.2 WebFlux vs MVC
```
Spring MVC:
  Thread-per-request model
  Blocking I/O
  Limited by thread pool size
  Easy to reason about
  
  Good for:
    - Traditional CRUD
    - Request processing fast
    - Simple applications

Spring WebFlux:
  Event-loop model
  Non-blocking I/O
  Few threads, many requests
  Steeper learning curve
  
  Good for:
    - High concurrency
    - Mostly I/O wait
    - Streaming
    - Real-time updates
```

### 5.3 WebFlux Controller
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    // Traditional MVC
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);  // Blocking
    }
    
    // WebFlux
    @GetMapping("/{id}")
    public Mono<User> getUser(@PathVariable Long id) {
        return userService.findById(id)  // Returns Mono (0 or 1)
            .subscribeOn(Schedulers.parallel())  // Thread to use
            .map(user -> enrich(user))           // Transform
            .onErrorResume(error -> ...);         // Handle errors
    }
}

Mono: 0 or 1 value
Flux: 0 to many values

Both are lazy:
  - Created but not executed
  - Execute only when subscribed
```

### 5.4 Backpressure
```
Problem: Producer too fast for consumer

Without backpressure:
  Producer: 1,000,000 events/sec
  Consumer: 10,000 events/sec
  
  Queue grows unbounded
  Memory exhausted
  System crashes

With backpressure:
  Consumer tells producer: "Send only 100 at a time"
  Producer respects: "OK, sending 100"
  Consumer processes 100
  Consumer: "Ready for next 100"
  Producer: "OK, sending next 100"
  
  Result: Controlled flow
```

---

## PART 6: REQUEST-SCOPED BEANS

### 6.1 Request Context
```
RequestContextHolder: ThreadLocal container for request

Thread 1 (request 1):
  RequestContextHolder.setRequestAttributes(attrs1)
  
Thread 2 (request 2):
  RequestContextHolder.setRequestAttributes(attrs2)

Each thread sees different attributes
No synchronization needed
```

### 6.2 Example: Audit Logging
```java
@Component
@Scope("request")
public class AuditContext {
    private String userId;
    private String correlationId;
    
    @PostConstruct
    public void initialize() {
        HttpServletRequest request = 
            ((ServletRequestAttributes) 
                RequestContextHolder.getRequestAttributes())
            .getRequest();
        
        userId = extractFromToken(request);
        correlationId = request.getHeader("X-Correlation-ID");
    }
}

Usage:

@Service
public class UserService {
    @Autowired
    private AuditContext auditContext;
    
    public void createUser(User user) {
        user.setCreatedBy(auditContext.getUserId());
        userRepository.save(user);
        
        logger.info("User created by: " + 
                    auditContext.getUserId() + 
                    " correlation: " + 
                    auditContext.getCorrelationId());
    }
}

Benefits:
  - No parameter passing needed
  - Context automatically available
  - Per-request isolation
  - Clean code
```

---

## PART 7: TRANSACTION MANAGEMENT

### 7.1 @Transactional
```java
@Service
public class TransferService {
    
    @Transactional
    public void transfer(Account from, Account to, int amount) {
        from.setBalance(from.getBalance() - amount);
        accountRepo.save(from);
        
        to.setBalance(to.getBalance() + amount);
        accountRepo.save(to);
        
        // If exception here: both operations rollback
    }
}

What happens:
  1. Spring wraps method in transaction
  2. Database connection obtained
  3. Transaction begins
  4. Method executes
  5. If no exception: commit (data persisted)
  6. If exception: rollback (data discarded)
  7. Connection returned to pool
```

### 7.2 Transaction Internals
```
Spring uses Proxies for @Transactional:

Caller
  ↓
Proxy (created by Spring AOP)
  ↓
Begin transaction (via PlatformTransactionManager)
  ↓
Actual method executes
  ↓
No exception?
  ├─ YES: commit()
  └─ NO: rollback()
  ↓
Proxy returns to caller

Database operations:
  BEGIN;
  INSERT INTO accounts...
  UPDATE accounts...
  COMMIT;
  
Or on error:
  BEGIN;
  INSERT INTO accounts...
  UPDATE accounts...  ← fails
  ROLLBACK;  ← undoes INSERT too
```

### 7.3 Isolation Levels
```
READ_UNCOMMITTED: No locks (DANGEROUS)
  T1: writes B=100
  T2: reads B=100 (uncommitted!)
  T1: rolls back B=50
  T2: has stale data

READ_COMMITTED: Read locks (DEFAULT)
  T1: locks B, writes B=100
  T2: waits for T1 to commit
  T2: reads B=100 (committed value)

REPEATABLE_READ: Row locks
  T1: reads B, locks B
  T2: waits for T1
  T1: reads B again, gets same value

SERIALIZABLE: Strictest (slowest)
  T1: runs to completion
  T2: runs only after T1 done
  Complete isolation
```

---

## PART 8: COMMON PRODUCTION PATTERNS

### 8.1 Service Layer Pattern
```java
// Controller (HTTP layer)
@RestController
public class UserController {
    @Autowired
    private UserService userService;
    
    @PostMapping("/users")
    public User createUser(@RequestBody User user) {
        return userService.create(user);
    }
}

// Service (Business logic)
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    @Transactional
    public User create(User user) {
        validateUser(user);
        User saved = userRepository.save(user);
        publishUserCreatedEvent(saved);
        return saved;
    }
}

// Repository (Data access)
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    User findByEmail(String email);
}

Separation of Concerns:
  - Controller: HTTP mapping
  - Service: Business logic
  - Repository: Data access
  - Each layer independently testable
```

### 8.2 Error Handling
```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UserNotFoundException ex) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getMessage()));
    }
    
    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            ValidationException ex) {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse(ex.getMessage()));
    }
}

Benefits:
  - Centralized error handling
  - Consistent error responses
  - No try-catch everywhere
```

---

## PART 9: PRODUCTION CONSIDERATIONS

### 9.1 Singleton Thread Safety
```
Danger:

@Service
public class UserService {
    private List<String> cache = new ArrayList<>();  // DANGER!
    
    public void addToCache(String item) {
        cache.add(item);  // Multiple threads!
    }
}

Fix:

@Service
public class UserService {
    private List<String> cache = 
        Collections.synchronizedList(new ArrayList<>());
    
    // OR use ConcurrentHashMap
    
    // OR make immutable
    private final List<String> immutableCache = 
        Collections.unmodifiableList(...);
}
```

### 9.2 Bean Initialization Order
```
Sometimes beans depend on startup order:

@Component
public class DatabaseInitializer {
    @Autowired
    private UserRepository userRepository;
    
    @PostConstruct
    public void initialize() {
        // Runs after all dependencies injected
        // Runs before application ready
        if(userRepository.count() == 0) {
            loadDefaultUsers();
        }
    }
}

Order:
  1. All beans created
  2. All dependencies injected
  3. @PostConstruct methods called (in arbitrary order)
  4. ApplicationReadyEvent fired
  5. Application starts accepting requests
```

### 9.3 Circular Dependency
```
Problem:

@Service
public class UserService {
    @Autowired
    private OrderService orderService;  // Depends on Order
}

@Service
public class OrderService {
    @Autowired
    private UserService userService;    // Depends on User
}

Spring detects: Circular dependency!

Solution:
  - Use @Lazy injection (delays resolution)
  - Use ObjectProvider (late binding)
  - Refactor code (extract third service)
```

---

## PART 10: DEBUGGING SPRING

### 10.1 Common Issues
```
Issue: Bean not found
  @Autowired
  private UnknownBean bean;
  
Solution:
  - Check @Component / @Service annotation
  - Check @ComponentScan path
  - Check bean name matches

Issue: Wrong instance injected
  Multiple implementations of interface
  
Solution:
  - Use @Qualifier("beanName")
  - Use @Primary
  - Multiple beans with different qualifiers

Issue: AOP not working
  @Transactional not triggering
  
Solution:
  - Check method not private (AOP uses proxies)
  - Check transaction manager configured
  - Check @EnableTransactionManagement present

Issue: RequestScope outside request
  RequestContextHolder.getRequestAttributes() is null
  
Solution:
  - Called outside request context
  - Move to @Controller method or use RequestScope bean
```

### 10.2 Performance Issues
```
Problem: Slow startup
  Spring loading too many beans
  
Profiling:
  -Dspring.jpa.properties.hibernate.generate_statistics=true
  
Solution:
  - Lazy initialization
  - Remove unused dependencies
  - Use conditional beans

Problem: High latency
  Spring AOP creating too many proxies
  
Solution:
  - Reduce @Transactional usage
  - Use faster aspect implementation
  - Profile with JProfiler
```

---

## KEY TAKEAWAYS

1. **Spring IoC manages bean lifecycle** (creation, injection, cleanup)
2. **Constructor injection is preferred** (explicit, testable)
3. **Singletons must be thread-safe** (multiple threads access)
4. **AOP uses proxies** (intercept method calls)
5. **@Transactional uses proxies too** (rollback on exception)
6. **Request-scoped beans isolated per request** (ThreadLocal)
7. **WebFlux for high concurrency** (event loop, non-blocking)
8. **Security stored in ThreadLocal** (per-request, no sync needed)
9. **Circular dependencies prevented** (Spring detects early)
10. **Profile and debug with Spring Boot Actuator**

---

*This KB covers Spring framework concepts needed for production systems and interview discussions.*
