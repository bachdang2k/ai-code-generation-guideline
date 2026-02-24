# RULE PHÁT TRIỂN HỆ THỐNG QUẢN LÝ THU HỘ BHXH

## 1. TỔNG QUAN HỆ THỐNG

- **Web Admin**: Đại lý thu hộ + nhân viên thu hộ (responsive UI)
- **Java Backend**: REST API nghiệp vụ (vn.vivas.bh)
- **Tech Stack**: Spring Boot 3.4.4, Java 21, MySQL 8, Redis, Flyway
- **Language**: Tiếng Việt (biến, class, comment, business logic)
- **Auth**: JWT decode, @PreAuthorize permissions
- **API Base**: `/api/v1/**` (CMS/portal), `/api/app/v1/**` (App)

---

## 2. QUY TẮC DATABASE (BẮT BUỘC)

❌ **KHÔNG**:
- Tạo migration files mới
- Thay đổi Entity JPA (thêm/xóa cột/bảng)
- Chạy script DDL (CREATE/ALTER/DROP TABLE)

✅ **ĐƯỢC**:
- Gợi ý thiết kế DB
- Viết code Java (Entity, Repository, Service)
- Tạo seed data (INSERT scripts)

**Lý do**: Database được quản lý bởi team riêng.

---

## 4. JAVA BACKEND

**Package**: `vn.vivas.bh`
**Build**: `./mvnw clean package`
**Run**: `./mvnw spring-boot:run`
**Test**: `./mvnw test` (unit), `./mvnw verify` (all)

**Config**: application.yml (MySQL, Hibernate ddl-auto=validate, Flyway, Swagger)

---

## 4.0. API PAGINATION

- Request: `vn.vivas.bh.request.PaginationRequest`
- Response: `vn.vivas.bh.response.PageResponse`

---

## 4.1. ERROR HANDLING & RESPONSE

### MessageResponseDict Pattern
✅ **BẮT BUỘC**: All response codes in `MessageResponseDict` enum

**Format**:
```java
// Success: code=0
ADD_SUCCESS(HttpStatus.OK, 0, "Thêm mới thành công"),
UPDATE_SUCCESS(HttpStatus.OK, 0, "Cập nhật thành công"),
DELETE_SUCCESS(HttpStatus.OK, 0, "Xóa thành công"),
SUCCESS(HttpStatus.OK, 0, "Thành công"),

// Errors: code=100000-400000
HO_SO_NOT_FOUND(HttpStatus.NOT_FOUND, 100001, "Hồ sơ không tồn tại"),
INVALID_PHONE(HttpStatus.OK, 100002, "Số điện thoại không hợp lệ"),
OBJECT_NOT_FOUND(HttpStatus.NOT_FOUND, 100003, "%s không tồn tại"),  // dynamic %s
CANNOT_DELETE_CONSTRAINT(HttpStatus.OK, 100004, "Không thể xóa %s vì ràng buộc"),
```

### Usage
```java
// Simple
throw new BusinessException(MessageResponseDict.HO_SO_NOT_FOUND);

// With dynamic params
throw new BusinessException(MessageResponseDict.OBJECT_NOT_FOUND, List.of("Đại lý"));
throw new BusinessException(MessageResponseDict.CANNOT_DELETE_CONSTRAINT, List.of("Hồ sơ"));

// Response
return ResponseUtils.ok(MessageResponseDict.ADD_SUCCESS, data);
return ResponseEntity.ok(new ResponseCommon<>(
    MessageResponseDict.SUCCESS.getCode(),
    MessageResponseDict.SUCCESS.getMessage(),
    data
));

// ❌ NEVER hardcode
// throw new BusinessException("Not found");
// return ResponseEntity.badRequest().body("Error");
```

### Checklist
- [ ] All codes in enum (no hardcode)
- [ ] Success = 0, Error = 100000-400000
- [ ] HttpStatus = OK (except NOT_FOUND, UNAUTHORIZED)
- [ ] Use `List<String>` for dynamic params
- [ ] Throw BusinessException, don't return error codes

---

## 4.2. LOGGING

### Format
```java
log.info("[{userId}/{action}] id={id}, status={status}", userId, "CREATE", id, status);
```

### Levels
- `error`: System errors with stacktrace → `log.error("Msg: {}", ex.getMessage(), ex)`
- `warn`: Business errors → `log.warn("Code={} Msg={}", code, msg)`
- `info`: Business operations, state changes
- `debug`: Details (disabled in prod)

### Rules
- Log START/END of transactions
- Log state changes (status updates)
- NO sensitive data (passwords, tokens)
- Loop items: log summary count, not each item
- Failed operations: log full context

---

## 4.3. SERVICE LAYER

### Method Pattern
```java
public Response methodName(Request request) {
    // 1. Auth
    AccountInfo user = AuthenticationService.getCurrentUserRequired();

    // 2. Validate
    validateInput(request);

    // 3. Load
    Entity entity = repository.findById(id)
        .orElseThrow(() -> new BusinessException(MessageResponseDict.NOT_FOUND));

    // 4. Logic
    entity.setState(newState);

    // 5. Persist
    repository.save(entity);

    // 6. Return
    return mappingHelper.map(entity, Response.class);
}
```

### Validation
```java
private void validateInput(Request request) {
    if (StringUtils.isEmpty(request.getName()))
        throw new BusinessException(MessageResponseDict.FIELD_REQUIRED);

    if (!isValidPhone(request.getPhone()))
        throw new BusinessException(MessageResponseDict.INVALID_PHONE);
}
```

### Transaction
- `@Transactional` for multi-DB operations
- Validate BEFORE transaction block
- Use `TransactionTemplate` for programmatic control:
  ```java
  transactionTemplate.executeWithoutResult(status -> {
      // DB operations
  });
  ```

---

## 4.4. CODING CONVENTIONS

📖 **See**: `AGENTS.md` for complete conventions

**Quick ref**:
- Methods: `verbNoun()` in Vietnamese (e.g., `timKiem()`, `taoHoSo()`)
- Classes: `PascalCase` (e.g., `HoSoService`)
- DB: `snake_case`, Java: `camelCase`
- Constants: `UPPER_SNAKE_CASE`
- Booleans: `isXxx()`, `hasXxx()` (not `getXxx()`)

---

## 4.5. SECURITY & AUTHORIZATION

### Authentication
```java
AccountInfo user = AuthenticationService.getCurrentUserRequired();
```

### ABAC (Hierarchical Access Control)
**Pattern** - Admin/Agency data filtering:

```java
//Logic build userDetail role = ADMIN (SYSADMIN have userId = -1)
roleCustom = new SimpleGrantedAuthority("ROLE_" + Role.ADMIN);
authorities.add(roleCustom);
String fullName = user.getFullName() == null ? "SYS_ADMIN" : user.getFullName();

        return new UserDetailsCustom(
        user.getId(),
                user.getUsername(),
                null, user.getEmail(),
                user.getFullName(),
                user.getType(),
                null, null, user.getAgency() == null ? -1 : user.getAgency().getId(),
                user.getAgency() == null ? fullName : user.getAgency().getName(), authorities);
        
//--- 
UserContext userLogin = UserContext.getCurrentUser();
List<Integer> listAgentID = List.of();

if (userLogin.getRole().equals(Role.ADMIN)) {
    Integer daiLyLoginId = userLogin.getDaiLyId() > -1 ? userLogin.getDaiLyId() : null;
    Integer daiLyFilterId = daiLyThuId == null ? daiLyLoginId : daiLyThuId;
    if (daiLyFilterId != null) {
        listAgentID = agentRepository.findChildrenById(daiLyFilterId);
        listAgentID.add(daiLyFilterId);
    }
}

PageResponse<ThongTinHoSoDto> result = hoSoService.timKiem(
    keyword, loaiNghiepVu, maTrangThai, tuNgay, denNgay,
    listAgentID, nhanVienThuId, pageable
);
```

### Rules
- **ADMIN (Role ADMIN - userId = -1) no filter**: `listAgentID = empty` → see ALL data
- **ADMIN (Role ADMIN - userId = -1) with filter**: `listAgentID = filterId + children` → see filtered hierarchy
- **AGENCY user (Role ADMIN - userId > -1)**: `listAgentID = ownId + children` → see own hierarchy only
- **Pattern**: Always use `agentRepository.findChildrenById(id)` for hierarchical lookup
- **Apply to**: All queries returning agency-filtered data (Hồ sơ, Đợt kê khai, etc.)



---

## 4.6. TESTING

### Structure
- **Unit tests**: `*Test.java` (e.g., `HoSoServiceTest`)
  - `@ExtendWith(MockitoExtension.class)`
  - `@Mock` + `@InjectMocks`
  - Fast, isolated business logic tests

- **Integration tests**: `*IT.java` (e.g., `HoSoRepositoryIT`)
  - `@SpringBootTest` + `@Import(TestcontainersConfiguration.class)`
  - Real MySQL database with TestContainers
  - Test complete flows

### Test Auth Setup (All Tests)

**Module doesn't manage Auth - always fake UserDetail**:

```java
@BeforeEach
void setUp() {
    AccountInfo adminUser = AccountInfo.builder()
        .id(1)
        .role(Role.ADMIN)
        .permissions(Arrays.asList("ALL_PERMISSIONS"))
        .daiLyId(null)  // Full data access
        .build();

    when(AuthenticationService.getCurrentUserRequired()).thenReturn(adminUser);
    when(UserContext.getCurrentUser()).thenReturn(adminUser);
}
```

**Apply to**: All unit tests and integration tests

### Quick Examples
```java
// Unit test
@ExtendWith(MockitoExtension.class)
class HoSoServiceTest {
    @Mock private HoSoRepository repo;
    @InjectMocks private HoSoService service;

    @Test void shouldTaoHoSuWhenValidInput() {
        // Given
        when(repo.save(any())).thenReturn(hoSo);
        // When
        HoSo result = service.taoHoSo(request);
        // Then
        assertThat(result.getMaHoSo()).isEqualTo("HS001");
        verify(repo, times(1)).save(any());
    }
}

// Integration test
@SpringBootTest
@Import(TestcontainersConfiguration.class)
class HoSoRepositoryIT {
    @Autowired private HoSoRepository repo;

    @BeforeEach void setUp() { repo.deleteAll(); }

    @Test void shouldSaveAndFind() {
        HoSo saved = repo.save(hoSo);
        assertThat(repo.findById(saved.getId())).isPresent();
    }
}
```

### Key Rules
- Vietnamese names: `shouldTaoHoSuWhenValidInput()`
- Clean DB in `@BeforeEach`
- TestContainers config: package-private (no `public` modifier)
- Don't test framework code or getters/setters

### Running Tests
```bash
./mvnw test                           # Unit tests only
./mvnw verify                         # All tests (unit + integration)
./mvnw failsafe:integration-test      # Integration tests only
./mvnw test -Dtest=HoSoServiceTest    # Specific test class
```

📖 **Complete guide**: `references/TEST.md`
  - TestContainers 2.0 setup with MySQL 8
  - Controller tests with `@WebMvcTest`
  - Spring Boot 4 migration (`@MockitoBean`)
  - Best practices and naming conventions

---

## 5. EXTERNAL REFERENCES

📖 **Detailed guides**:
- `AGENTS.md` - Complete coding conventions (Lombok, MapStruct, patterns)
- `references/TEST.md` - Testing best practices

---

**Last Updated**: 2025-02-21
**Version**: 2.0 (Optimized)
