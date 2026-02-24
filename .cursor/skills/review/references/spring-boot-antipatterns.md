# Spring Boot Common Antipatterns and Fixes

This document catalogs common mistakes in Spring Boot applications and their solutions.

## 1. Transaction Management Issues

### ❌ Antipattern 1.1: Missing @Transactional
```java
// Multiple DB operations without transaction
public void updateUser(Long userId, UserDTO dto) {
    User user = userRepository.findById(userId).get();
    user.setName(dto.getName());
    userRepository.save(user);
    
    auditRepository.save(new Audit(userId, "UPDATE"));
    // If audit save fails, user is already saved - inconsistent state
}
```

**Fix:**
```java
@Transactional
public void updateUser(Long userId, UserDTO dto) {
    User user = userRepository.findById(userId).get();
    user.setName(dto.getName());
    userRepository.save(user);
    
    auditRepository.save(new Audit(userId, "UPDATE"));
    // Now both save or both rollback
}
```

### ❌ Antipattern 1.2: Self-Invocation of @Transactional
```java
@Service
public class UserService {
    public void publicMethod() {
        privateTransactionalMethod(); // Won't work! No proxy
    }
    
    @Transactional
    private void privateTransactionalMethod() {
        // Transaction won't start
    }
}
```

**Fix:**
```java
@Service
public class UserService {
    @Autowired
    private UserService self; // Inject self
    
    public void publicMethod() {
        self.privateTransactionalMethod(); // Works through proxy
    }
    
    @Transactional
    public void privateTransactionalMethod() {
        // Transaction works now
    }
}
```

### ❌ Antipattern 1.3: Wrong Propagation Level
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void updateMultipleUsers(List<Long> userIds) {
    for (Long id : userIds) {
        updateSingleUser(id); // Each gets new transaction
    }
    // Can't rollback all if one fails
}
```

**Fix:**
```java
@Transactional // Default REQUIRED propagation
public void updateMultipleUsers(List<Long> userIds) {
    for (Long id : userIds) {
        updateSingleUser(id); // Same transaction
    }
    // All-or-nothing
}
```

## 2. JPA/Entity Issues

### ❌ Antipattern 2.1: N+1 Query Problem
```java
List<User> users = userRepository.findAll(); // 1 query
for (User user : users) {
    List<Order> orders = user.getOrders(); // N queries!
    orders.size();
}
```

**Fix Option 1: @EntityGraph**
```java
public interface UserRepository extends JpaRepository<User, Long> {
    @EntityGraph(attributePaths = {"orders"})
    List<User> findAll();
}
```

**Fix Option 2: JOIN FETCH**
```java
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

### ❌ Antipattern 2.2: Lazy Loading Outside Transaction
```java
public UserDTO getUser(Long id) {
    User user = userRepository.findById(id).get();
    return mapper.toDTO(user); // Transaction ends here
}

public class UserDTO {
    // ...
    public List<OrderDTO> getOrders() {
        return user.getOrders(); // LazyInitializationException!
    }
}
```

**Fix:**
```java
@Transactional(readOnly = true)
public UserDTO getUser(Long id) {
    User user = userRepository.findById(id).get();
    user.getOrders().size(); // Force load within transaction
    return mapper.toDTO(user);
}
```

### ❌ Antipattern 2.3: Bidirectional Relationship Infinite Loop
```java
@Entity
public class User {
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}

@Entity
public class Order {
    @ManyToOne
    private User user;
}

// JSON serialization causes infinite loop: User → Orders → User → ...
```

**Fix:**
```java
@Entity
public class User {
    @OneToMany(mappedBy = "user")
    @JsonIgnore // or @JsonManagedReference
    private List<Order> orders;
}

@Entity
public class Order {
    @ManyToOne
    @JsonBackReference
    private User user;
}
```

### ❌ Antipattern 2.4: Missing equals() and hashCode()
```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;
    // No equals/hashCode
}

Set<User> users = new HashSet<>();
users.add(user);
users.add(user); // Might add twice if entities detached/merged
```

**Fix:**
```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User user = (User) o;
        return id != null && id.equals(user.id);
    }
    
    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

### ❌ Antipattern 2.5: Using Entities as DTOs
```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).get(); // Exposes entity directly
}
```

**Fix:**
```java
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    User user = userRepository.findById(id).get();
    return mapper.toResponse(user); // Use DTO
}
```

## 3. Exception Handling Issues

### ❌ Antipattern 3.1: Swallowing Exceptions
```java
try {
    riskyOperation();
} catch (Exception e) {
    // Silent failure
}
```

**Fix:**
```java
try {
    riskyOperation();
} catch (Exception e) {
    log.error("Failed to perform risky operation", e);
    throw new BusinessException(MessageResponseDict.OPERATION_FAILED);
}
```

### ❌ Antipattern 3.2: Generic Exception Catching
```java
try {
    processPayment();
} catch (Exception e) { // Too broad
    return "Error occurred";
}
```

**Fix:**
```java
try {
    processPayment();
} catch (PaymentException e) {
    log.warn("Payment failed: {}", e.getMessage());
    throw new BusinessException(MessageResponseDict.PAYMENT_FAILED);
} catch (NetworkException e) {
    log.error("Network error during payment", e);
    throw new BusinessException(MessageResponseDict.NETWORK_ERROR);
}
```

### ❌ Antipattern 3.3: No @ControllerAdvice
```java
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        try {
            return userService.getUser(id);
        } catch (UserNotFoundException e) {
            return ResponseEntity.notFound().build(); // Repeated in every controller
        }
    }
}
```

**Fix:**
```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ResponseCommon<?>> handleBusinessException(BusinessException ex) {
        return ResponseEntity.status(ex.getHttpStatus())
            .body(new ResponseCommon<>(ex.getCode(), ex.getMessage(), null));
    }
}

@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUser(id); // Clean controller
    }
}
```

## 4. REST API Issues

### ❌ Antipattern 4.1: Wrong HTTP Method
```java
@GetMapping("/users/delete/{id}") // GET for mutation!
public void deleteUser(@PathVariable Long id) {
    userRepository.deleteById(id);
}
```

**Fix:**
```java
@DeleteMapping("/users/{id}")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userRepository.deleteById(id);
    return ResponseEntity.noContent().build();
}
```

### ❌ Antipattern 4.2: Missing Validation
```java
@PostMapping("/users")
public User createUser(@RequestBody UserRequest request) {
    return userService.create(request); // No validation
}
```

**Fix:**
```java
@PostMapping("/users")
public User createUser(@Valid @RequestBody UserRequest request) {
    return userService.create(request);
}

public class UserRequest {
    @NotBlank(message = "Name is required")
    private String name;
    
    @Email(message = "Invalid email")
    private String email;
    
    @Min(value = 18, message = "Must be 18+")
    private Integer age;
}
```

### ❌ Antipattern 4.3: No Pagination for Lists
```java
@GetMapping("/users")
public List<User> getAllUsers() {
    return userRepository.findAll(); // Returns all records!
}
```

**Fix:**
```java
@GetMapping("/users")
public Page<UserResponse> getAllUsers(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size
) {
    Pageable pageable = PageRequest.of(page, size);
    return userRepository.findAll(pageable)
        .map(mapper::toResponse);
}
```

### ❌ Antipattern 4.4: Inconsistent Status Codes
```java
@PostMapping("/users")
public User createUser(@RequestBody UserRequest request) {
    return userService.create(request); // Returns 200 instead of 201
}
```

**Fix:**
```java
@PostMapping("/users")
@ResponseStatus(HttpStatus.CREATED)
public User createUser(@RequestBody UserRequest request) {
    return userService.create(request);
}
```

## 5. Performance Issues

### ❌ Antipattern 5.1: SELECT * Instead of Specific Columns
```java
@Query("SELECT u FROM User u WHERE u.email = :email")
User findByEmail(String email); // Loads all columns
```

**Fix:**
```java
@Query("SELECT new com.example.UserBasicInfo(u.id, u.name, u.email) " +
       "FROM User u WHERE u.email = :email")
UserBasicInfo findBasicInfoByEmail(String email);
```

### ❌ Antipattern 5.2: Synchronous API Calls in Loop
```java
public void notifyUsers(List<User> users) {
    for (User user : users) {
        emailService.sendEmail(user.getEmail()); // Blocks for each
    }
}
```

**Fix:**
```java
@Async
public CompletableFuture<Void> sendEmailAsync(String email) {
    emailService.sendEmail(email);
    return CompletableFuture.completedFuture(null);
}

public void notifyUsers(List<User> users) {
    List<CompletableFuture<Void>> futures = users.stream()
        .map(user -> sendEmailAsync(user.getEmail()))
        .collect(Collectors.toList());
    
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
}
```

### ❌ Antipattern 5.3: No Caching for Frequently Read Data
```java
public List<Category> getCategories() {
    return categoryRepository.findAll(); // Query DB every time
}
```

**Fix:**
```java
@Cacheable("categories")
public List<Category> getCategories() {
    return categoryRepository.findAll();
}

@CacheEvict(value = "categories", allEntries = true)
public void updateCategory(Category category) {
    categoryRepository.save(category);
}
```

## 6. Security Issues

### ❌ Antipattern 6.1: SQL Injection via String Concatenation
```java
@Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)
List<User> findByName(String name); // SQL injection!
```

**Fix:**
```java
@Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
List<User> findByName(@Param("name") String name);
```

### ❌ Antipattern 6.2: Logging Sensitive Data
```java
log.info("User login: username={}, password={}", username, password); // Logs password!
```

**Fix:**
```java
log.info("User login attempt: username={}", username);
// Never log passwords, tokens, credit cards, etc.
```

### ❌ Antipattern 6.3: Missing Authorization Check
```java
@DeleteMapping("/users/{id}")
public void deleteUser(@PathVariable Long id) {
    userRepository.deleteById(id); // Anyone can delete any user!
}
```

**Fix:**
```java
@DeleteMapping("/users/{id}")
@PreAuthorize("hasRole('ADMIN') or @userSecurity.isOwner(#id)")
public void deleteUser(@PathVariable Long id) {
    userRepository.deleteById(id);
}
```

## Detection Checklist

When reviewing code, check for:
- [ ] All multi-operation methods have @Transactional
- [ ] No self-invocation of @Transactional methods
- [ ] No lazy loading outside transactions
- [ ] EntityGraph or JOIN FETCH used to avoid N+1
- [ ] Entities have proper equals/hashCode
- [ ] DTOs used instead of entities in responses
- [ ] @ControllerAdvice handles exceptions globally
- [ ] REST methods use correct HTTP verbs
- [ ] Input validation with @Valid and constraints
- [ ] List endpoints use pagination
- [ ] Proper HTTP status codes (201, 204, 404, etc.)
- [ ] No SELECT *, only needed columns
- [ ] Caching for frequently read data
- [ ] Async for independent operations
- [ ] No SQL injection (use @Param)
- [ ] No logging of sensitive data
- [ ] Authorization checks on all endpoints