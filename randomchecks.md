------------------------------
## 1. Spring Interceptors vs. Servlet Filters

* Filters: Part of the low-level Servlet Container (Tomcat) [Filter vs Interceptor StackOverflow]. They wrap around the DispatcherServlet and process raw HTTP inputs/outputs [Filter vs Interceptor StackOverflow]. They are ideal for universal, infrastructure-level tasks like CORS, global security, and encryption.
* Interceptors: Deeply embedded inside the Spring MVC application layer [Filter vs Interceptor StackOverflow]. They operate between the DispatcherServlet and your concrete controllers via preHandle(), postHandle(), and afterCompletion(). They understand Spring's architecture and are ideal for performance profiling, request auditing, multi-tenant setup, and UI model modifications [Filter vs Interceptor StackOverflow].

## 2. Spring Boot Lightweight Security & Custom Annotations

* You can craft manual token-based authentication and role-based authorization inside an Interceptor's preHandle() block by checking headers and querying custom annotations (like a homemade @RequiresRole) using Java Reflection.
* The Catch: Custom interceptor security does not protect against critical vulnerabilities (CSRF, Clickjacking) out-of-the-box like Spring Security does.

## 3. Spring Security Architecture

* Moves authentication and authorization completely away from Interceptors and places them into an early-stage Servlet Filter Chain [Filter vs Interceptor StackOverflow].
* Uses a declarative SecurityFilterChain bean to establish public and secure URL patterns.
* Replaces custom security annotations with robust, native Spring Expression Language (SpEL) annotations like @PreAuthorize [Spring Security PreAuthorize Annotation].

## 4. Advanced Algorithmic Concepts: Bloom Filters

* A highly space-efficient, probabilistic data structure used to verify set membership.
* The Absolute Rule: It returns "Definitely Not" (100% accurate, zero false negatives) or "Probably Yes" (potential false positives due to bit-collision).
* Works under the hood via a single Bit Array initialized to zeros and k independent hash functions.
* Widely used to shield databases (Cassandra, RocksDB) from expensive disk lookups for non-existent keys.

## 5. JVM Memory & Class Loading Lifecycle

* When executing a class, the JVM steps through Loading, Linking, and Initialization.
* Static Block Priority: During the Initialization phase, the compiler merges all static variables and static {} blocks into a hidden internal <clinit> (Class Initializer) method. The JVM runs this method before invoking the main method to guarantee the global environment is safe and configured.
* Process Isolation: This entire engine runs inside a single, dedicated OS sandboxed process. Static fields live in the Metaspace (per-class blueprint area) and survive for the entire lifespan of the process. They are shared across all CPU request execution threads, meaning state changes are instantly visible globally. Non-static instance fields live in the Heap and are bound to specific object instances created via new.

## 6. Java Keywords: static vs. final

* static: Dictates memory management. The field belongs directly to the class blueprint, ensuring there is exactly one copy in memory shared globally.
* final: Dictates immutability. It locks a variable's reference or value so it can never be reassigned after its initial definition.
* Combining them (static final) creates a strict, unchangeable global compile-time constant.

## 7. Internal HashMap Mechanics & Red-Black Trees

* Standard HashMap operates at O(1) time complexity but can degrade to linear O(N) during severe hash collisions where multiple keys map to the same bucket.
* Treeification: In Java 8+, if a single bucket's linked list hits a TREEIFY_THRESHOLD of 8 elements (and the overall map capacity is at least 64), Spring upgrades that bucket to a self-balancing Red-Black Tree to limit lookup time to a predictable $\log(N)$ threshold, natively protecting the app against Hash-Collision Denial of Service (DoS) attacks.
* Path Calculation: When you do not override hashCode(), Java generates a default HotSpot Identity Hash Code via the JVM. The HashMap performs a protective bit-shift mutation (h) ^ (h >>> 16) to distribute entropy, and maps the key to an array index using the highly efficient bitwise operation (Capacity - 1) & Hash.

## 8. Spring Bean Lifecycles: Singleton vs. Prototype

* Singleton (Default): Spring instantiates exactly one instance per application context [Singleton Beans Spring Framework]. Because individual incoming user requests are handled by distinct concurrent threads, Singleton beans must remain stateless to avoid thread-safety corruption [Singleton Beans Spring Framework].
* Prototype: Spring instantiates a brand-new instance on every single invocation request [Prototype Beans Spring Framework]. Once handed over, Spring relinquishes tracking control, and the object is cleaned up by the standard JVM Garbage Collector once references drop [Prototype Beans Spring Framework].
* ApplicationContext: Instantiated via SpringApplication.run(), which deduces your app environment type, prepares the concrete subclass blueprint, and executes refresh() to perform component scanning, parse definitions, and eagerly load singletons.

## 9. Spring AOP Proxies & The Self-Invocation Trap

* Annotations like @Around or @Transactional function via the Proxy Design Pattern [Filter vs Interceptor StackOverflow]. Spring creates a dynamic proxy wrapper (CGLIB) around your target bean [Spring Boot 3 default AOP proxy type].
* The Trap: Marking a huge class as @Transactional or executing an internal method call via the this reference triggers a Self-Invocation Failure. The local execution entirely bypasses the Spring dynamic proxy object, silently ignoring the transaction wrapper.
* The Production Fix: This is bypassed by refactoring the class into two separate, decoupled beans, executing self-injection via an @Autowired reference of the proxy itself, or managing transactions programmatically utilizing Spring's TransactionTemplate.

------------------------------
Whenever you want to pull SpringFactoriesLoader off the shelf to see how custom libraries plug in, or if you want to dive deeper into any of these low-level architectural blocks, just let me know! What part of your core system layout would you like to explore next?

