---
name: Root Cause Tracing for Java Backend
description: Systematically trace bugs backward through Spring Boot call stack and database operations to find original trigger
when_to_use: when errors occur deep in Spring Boot execution, JPA queries, or transaction management
version: 1.0.0
languages: Java version >= 21
frameworks: Spring Boot 3.x, JPA/Hibernate, MySQL
build_tool: Maven Wrapper
---

# Root Cause Tracing for Java Backend

## Overview

In Spring Boot applications, bugs often manifest deep in the execution stack:
- `ConstraintViolationException` in service layer
- `LazyInitializationException` during JSON serialization
- `DataIntegrityViolationException` when saving entities
- Wrong database connection in test execution
- Transactions committing with invalid data

**Core principle:** Trace backward through the call chain (controller → service → repository → database) until you find the original trigger, then fix at the source.

## When to Use

```
┌─────────────────────────────────────┐
│  Bug appears deep in stack?         │
│  (Service/Repository/Transaction)   │
└───────────┬─────────────────────────┘
            │ yes
            ▼
┌─────────────────────────────────────┐
│  Can trace backwards through:       │
│  • Stack traces                     │
│  • Spring AOP layers                │
│  • Transaction boundaries           │
│  • JPA/Hibernate logs               │
└───────────┬─────────────────────────┘
            │ yes
            ▼
┌─────────────────────────────────────┐
│  Trace to original trigger          │
│  Fix at source + add defense layers │
└─────────────────────────────────────┘
```

**Use when:**
- Exception thrown from deep in service/repository layer
- Database constraint violations from unclear origins
- Transaction issues (wrong isolation, premature commit)
- JPA lazy loading failures
- Test database pollution between test methods
- Wrong configuration applied in specific contexts

## The Tracing Process for Spring Boot

### 1. Observe the Symptom

```
org.hibernate.exception.ConstraintViolationException: 
  could not execute statement [Duplicate entry '123' for key 'users.email']
    at UserRepository.save(User.java:45)
```

### 2. Find Immediate Cause

**What code directly causes this?**

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // Called from service layer
}

// In service:
userRepository.save(user); // ← Exception thrown here
```

### 3. Ask: What Called This?

Trace the call chain through Spring layers:

```java
UserController.createUser()           // Layer 1: Web/Controller
  → UserService.registerUser()        // Layer 2: Business Logic
    → UserValidator.validate()        // Layer 3: Validation (missed duplicate!)
      → UserRepository.save()         // Layer 4: Persistence (FAILS HERE)
        → Hibernate/JPA                // Layer 5: ORM
          → MySQL Driver               // Layer 6: Database
```

### 4. Keep Tracing Up - Check Data Flow

**What value was passed?**

```java
// Controller receives DTO
@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(@RequestBody UserRequest request) {
    log.debug("Received request: {}", request); // ← Add this
    return userService.registerUser(request);
}

// Service transforms and saves
@Service
@Transactional
public class UserService {
    public UserResponse registerUser(UserRequest request) {
        log.debug("Processing user: email={}", request.getEmail()); // ← Add this
        
        // BUG: No duplicate check before save!
        User user = userMapper.toEntity(request);
        return userRepository.save(user);
    }
}
```

### 5. Find Original Trigger

**Where did duplicate email come from?**

Common root causes in Spring Boot:
- **Missing validation:** No `@UniqueEmail` or repository check
- **Race condition:** Two requests processed simultaneously
- **Test pollution:** Previous test left data in database
- **Wrong transaction boundary:** Read-check-write not atomic
- **Improper exception handling:** Retry logic causing duplicates

## Adding Instrumentation for Spring Boot

### Debug Logging Pattern

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class UserService {
    private static final Logger log = LoggerFactory.getLogger(UserService.class);
    
    @Transactional
    public UserResponse registerUser(UserRequest request) {
        // BEFORE the operation - log context
        log.debug("DEBUG registerUser: email={}, transaction={}, thread={}", 
            request.getEmail(),
            TransactionSynchronizationManager.getCurrentTransactionName(),
            Thread.currentThread().getName()
        );
        
        // Log call stack if needed
        if (log.isTraceEnabled()) {
            log.trace("Call stack: ", new Exception("TRACE"));
        }
        
        // The problematic operation
        User user = userMapper.toEntity(request);
        User saved = userRepository.save(user);
        
        log.debug("DEBUG saved user: id={}", saved.getId());
        return userMapper.toResponse(saved);
    }
}
```

**Critical for tests:** Use `@Slf4j` with proper `application-test.yml`:

```yaml
logging:
  level:
    com.yourcompany: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
    org.springframework.transaction: DEBUG
```

### Database Query Tracing

Enable detailed Hibernate logging:

```yaml
# application-test.yml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true
        generate_statistics: true
        
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type: TRACE  # Shows parameter binding
    org.hibernate.stat: DEBUG  # Shows statistics
```

### Transaction Boundary Tracing

```java
@Aspect
@Component
@Slf4j
public class TransactionTracingAspect {
    
    @Around("@annotation(org.springframework.transaction.annotation.Transactional)")
    public Object traceTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().toShortString();
        
        log.debug("TRANSACTION START: method={}, isolation={}, propagation={}", 
            methodName,
            TransactionSynchronizationManager.getCurrentTransactionIsolationLevel(),
            TransactionSynchronizationManager.isActualTransactionActive()
        );
        
        try {
            Object result = joinPoint.proceed();
            log.debug("TRANSACTION SUCCESS: method={}", methodName);
            return result;
        } catch (Exception e) {
            log.error("TRANSACTION FAILED: method={}, error={}", methodName, e.getMessage());
            throw e;
        }
    }
}
```

## Finding Which Test Causes Database Pollution

### Problem: Test leaves data that affects other tests

```java
// Test 1 creates user with email "test@example.com"
@Test
void testCreateUser() {
    userService.registerUser(new UserRequest("test@example.com", "John"));
    // ❌ Doesn't clean up
}

// Test 2 fails due to duplicate
@Test
void testDuplicateEmail() {
    // ❌ Fails if Test 1 ran first
    userService.registerUser(new UserRequest("test@example.com", "Jane"));
}
```

### Solution 1: Proper Test Isolation

```java
@SpringBootTest
@Transactional  // ← Rollback after each test
@Sql(scripts = "/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
class UserServiceTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @BeforeEach
    void setUp() {
        // Clean slate for each test
        userRepository.deleteAll();
        
        // Verify clean state
        long count = userRepository.count();
        if (count != 0) {
            throw new IllegalStateException(
                "Database not clean! Found " + count + " users. " +
                "Check previous test cleanup. Thread: " + Thread.currentThread().getName()
            );
        }
    }
    
    @AfterEach
    void tearDown() {
        // Explicit cleanup as defense-in-depth
        userRepository.deleteAll();
    }
}
```

### Solution 2: Test Container Isolation

```java
@SpringBootTest
@Testcontainers
class UserServiceIntegrationTest {
    
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }
    
    // Each test gets isolated container state
}
```

### Solution 3: Find the Polluter

Create `find-polluter.sh`:

```bash
#!/bin/bash
# Usage: ./find-polluter.sh "SELECT COUNT(*) FROM users" "src/test/java/**/*Test.java"

QUERY="$1"
TEST_PATTERN="$2"

# Find all test files
TEST_FILES=$(find src/test/java -name "*Test.java" | sort)

for test_file in $TEST_FILES; do
    echo "Testing: $test_file"
    
    # Run single test
    ./mvnw test -Dtest=$(basename $test_file .java)
    
    # Check for pollution
    if ./mvnw exec:java -Dexec.mainClass="CheckDbState" -Dexec.args="$QUERY"; then
        echo "❌ POLLUTER FOUND: $test_file"
        exit 1
    fi
done

echo "✅ No polluter found"
```

## Real Example: Duplicate Key Violation

### Symptom
```
DataIntegrityViolationException: Duplicate entry 'john@example.com' for key 'users.email'
  at UserRepository.save() line 45
```

### Trace Chain

**Layer 6 (Database):** MySQL rejects duplicate email

**Layer 5 (JPA/Hibernate):** Hibernate executes INSERT without checking

**Layer 4 (Repository):** `userRepository.save(user)` called

**Layer 3 (Service):** No duplicate check before save
```java
@Transactional
public UserResponse registerUser(UserRequest request) {
    // ❌ BUG: Missing duplicate check
    User user = userMapper.toEntity(request);
    return userRepository.save(user);  // Fails here
}
```

**Layer 2 (Validation):** `@Valid` checks format but not uniqueness
```java
public record UserRequest(
    @Email String email,  // ✅ Validates format
    // ❌ Missing @UniqueEmail
    @NotBlank String name
) {}
```

**Layer 1 (Controller):** Accepts any valid-format email
```java
@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(@Valid @RequestBody UserRequest request) {
    return ResponseEntity.ok(userService.registerUser(request));
}
```

**Root Cause:** Missing uniqueness validation at service layer

### Fix at Source

```java
@Service
@Transactional
public class UserService {
    
    public UserResponse registerUser(UserRequest request) {
        // ✅ FIX: Check uniqueness at source
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException(request.getEmail());
        }
        
        User user = userMapper.toEntity(request);
        User saved = userRepository.save(user);
        return userMapper.toResponse(saved);
    }
}

// Repository method
public interface UserRepository extends JpaRepository<User, Long> {
    boolean existsByEmail(String email);
}
```

### Defense-in-Depth Layers

**Layer 1: Database Constraint**
```sql
ALTER TABLE users ADD UNIQUE INDEX idx_email (email);
```

**Layer 2: JPA Entity Constraint**
```java
@Entity
@Table(name = "users", uniqueConstraints = {
    @UniqueConstraint(columnNames = "email", name = "uk_user_email")
})
public class User {
    @Column(nullable = false, unique = true)
    private String email;
}
```

**Layer 3: Custom Validator**
```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        return email == null || !userRepository.existsByEmail(email);
    }
}
```

**Layer 4: Service-Level Check**
```java
@Transactional
public UserResponse registerUser(UserRequest request) {
    // Explicit check with better error message
    if (userRepository.existsByEmail(request.getEmail())) {
        log.warn("Duplicate registration attempt: email={}", request.getEmail());
        throw new DuplicateEmailException(
            "User with email " + request.getEmail() + " already exists"
        );
    }
    
    User user = userMapper.toEntity(request);
    User saved = userRepository.save(user);
    return userMapper.toResponse(saved);
}
```

**Layer 5: Optimistic Locking**
```java
@Entity
public class User {
    @Version
    private Long version;  // Prevents concurrent modifications
}
```

## Common Spring Boot Root Causes

### 1. Transaction Boundary Issues

**Symptom:** `LazyInitializationException` during JSON serialization

**Trace:**
```
Controller.getUser() 
  → Service.findUser() [@Transactional - ends here]
    → Repository.findById() 
      → Returns User with lazy-loaded Orders
  → Jackson serializes User
    → Tries to access orders ❌ Session closed!
```

**Root Cause:** Transaction ended before serialization

**Fix:**
```java
// Option 1: Eager fetch in query
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
Optional<User> findByIdWithOrders(@Param("id") Long id);

// Option 2: Open-in-View (use cautiously)
spring.jpa.open-in-view=true

// Option 3: DTO projection (recommended)
@Transactional(readOnly = true)
public UserResponse findUser(Long id) {
    User user = userRepository.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));
    return userMapper.toResponse(user); // Triggers fetch inside transaction
}
```

### 2. Test Transaction Propagation

**Symptom:** Test passes alone, fails in suite

**Trace:**
```
Test1.setup() [@Transactional]
  → Creates User (transaction not committed)
Test1.test() 
  → Finds user ✅ (same transaction)
  → Rollback

Test2.test()
  → Tries to find user ❌ Not found (rolled back)
```

**Root Cause:** `@Transactional` on test class causes rollback

**Fix:**
```java
@SpringBootTest
// Don't use @Transactional on class if tests need committed data
class UserServiceIntegrationTest {
    
    @Test
    void testUserCreation() {
        // Manually manage transactions when needed
        User user = transactionTemplate.execute(status -> {
            return userRepository.save(new User("test@example.com"));
        });
        
        // User is committed, available to other operations
        assertThat(userRepository.findById(user.getId())).isPresent();
    }
    
    @AfterEach
    void cleanup() {
        userRepository.deleteAll();
    }
}
```

### 3. Wrong DataSource in Tests

**Symptom:** Tests modify production database

**Trace:**
```
Test execution
  → Spring loads application.yml (production DB!)
  → No application-test.yml override
  → Connects to production MySQL
  → Test deletes data ❌
```

**Root Cause:** Missing test configuration

**Fix:**
```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/testdb
    username: test_user
    password: test_pass
  jpa:
    hibernate:
      ddl-auto: create-drop  # Fresh schema each run
  sql:
    init:
      mode: always
      
# Or use H2 in-memory
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
```

```java
@SpringBootTest
@ActiveProfiles("test")  // ← Force test profile
class UserServiceTest {
    // Tests use test database
}
```

## Key Principles

```
┌─────────────────────────────────┐
│  Found immediate cause          │
│  (Exception location)           │
└────────────┬────────────────────┘
             │
             ▼
      ┌─────────────┐
      │ Trace up    │ ◄─┐
      │ one layer   │   │
      └──┬──────────┘   │
         │              │
         ▼              │
   ┌──────────────┐    │
   │ Is this the  │    │
   │ source?      │    │
   └──┬───────────┘    │
      │                │
      ├─ No ───────────┘
      │
      └─ Yes
         │
         ▼
   ┌─────────────────────┐
   │ Fix at source       │
   │ + Add defense layers│
   └─────────────────────┘
```

**NEVER fix just where the exception is thrown.**

Trace back through:
1. Exception stack trace
2. Spring AOP proxies
3. Transaction boundaries
4. Service call chain
5. Controller/entry point
6. Test setup code

## Maven Commands for Debugging

```bash
# Run single test with debug logging
./mvnw test -Dtest=UserServiceTest -Dlogging.level.com.yourcompany=DEBUG

# Run with Hibernate SQL logging
./mvnw test -Dspring.jpa.show-sql=true

# Run with remote debugging
./mvnw test -Dmaven.surefire.debug

# Check for test pollution
./mvnw clean test -Dtest=UserServiceTest
./mvnw test -Dtest=SuspectTest  # Run without clean

# Generate detailed test report
./mvnw surefire-report:report
```

## Stack Trace Analysis Tips

**In Spring Boot:**
- Use `@Slf4j` with DEBUG level for tests
- Enable Hibernate SQL logging (`show-sql: true`)
- Add transaction tracing aspect
- Log before dangerous operations, not after

**Include in logs:**
- Method parameters
- Transaction status (`TransactionSynchronizationManager`)
- Thread name (for concurrent issues)
- Database connection info
- Entity state (managed/detached/transient)

**Capture context:**
```java
log.debug("DEBUG operation: param={}, txActive={}, thread={}, connection={}", 
    param,
    TransactionSynchronizationManager.isActualTransactionActive(),
    Thread.currentThread().getName(),
    DataSourceUtils.getConnection(dataSource)
);
```

## Real-World Impact

**Before:** `DataIntegrityViolationException` in production
- Fixed by adding try-catch in controller
- Root cause (missing validation) remained
- Same bug occurred with different data

**After Root Cause Tracing:**
1. Traced 5 layers back to missing validation
2. Fixed at source (service-level check)
3. Added 4 defense layers (DB, entity, validator, service)
4. Added test isolation (`@Transactional`, cleanup)
5. Zero duplicate errors in production

## Checklist

When debugging Spring Boot issues:

- [ ] Captured full stack trace with context
- [ ] Traced backward through all call layers
- [ ] Identified original trigger (controller/test/scheduler)
- [ ] Found where invalid data entered system
- [ ] Fixed at source, not at symptom
- [ ] Added validation at each layer
- [ ] Verified test isolation (no pollution)
- [ ] Checked transaction boundaries
- [ ] Enabled proper logging for reproduction
- [ ] Confirmed fix works in all scenarios

**Remember:** The exception location is rarely the bug location. Always trace to the source.