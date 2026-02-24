# BHxH Backend - Agent Development Guide

This guide is for agentic coding assistants working on the BHXH (Bảo hiểm xã hội) backend project.

## Project Overview

- **Stack**: Spring Boot 3.4.4, Java 21, Maven, MySQL 8, Flyway, Redis, OpenTelemetry
- **Purpose**: Hệ thống quản lý thu hộ BHXH tự nguyện và BHYT hộ gia đình (Voluntary BHXH and Family BHYT Collection System)
- **Package**: vn.vivas.bh
- **API Base**: `/api/v1/**` (CMS), `/api/app/v1/**` (Mobile App)

## Build, Test, and Run Commands

```bash
# Build the project
./mvnw clean package

# Run the application (dev)
./mvnw spring-boot:run

# Build without tests
./mvnw clean package -DskipTests

# Run specific test class
./mvnw test -Dtest=ClassName

# Run specific test method
./mvnw test -Dtest=ClassName#methodName

# Run with specific profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

**Note**: This project currently has no test files. When adding tests, use JUnit 5 and spring-boot-starter-test. Prioritize adding tests for new features.

## Code Style Guidelines

### Imports

Group imports: Java standard library (java.*, jakarta.*) → Third-party (org.*, com.*, lombok.*, io.*) → Project (vn.vivas.bh.*). No wildcard imports except static.

### Naming Conventions

- **Classes**: PascalCase (e.g., `HoSoService`, `NvHoSoDangKy`)
- **Methods**: camelCase, Vietnamese for business logic (e.g., `timKiem`, `kiNopHoSo`, `chuyenTrangThai`)
- **Variables**: camelCase, **Constants**: UPPER_SNAKE_CASE
- **Database**: snake_case tables/columns, camelCase entity fields mapped to snake_case via JPA

### Lombok Annotations

`@Data @Builder @NoArgsConstructor @AllArgsConstructor` (DTOs/Requests/Responses), `@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder` (Entities), `@RequiredArgsConstructor @Slf4j` (Services/Controllers), `@UtilityClass` (Utilities), `@FieldDefaults(level = AccessLevel.PRIVATE)` (Response DTOs).

### Entity Mapping

Use `@Table(name = "snake_case")` and `@Column(name = "snake_case")`. Use `@Enumerated(EnumType.STRING)` for enums. Use `@CreatedDate/@LastModifiedDate` with `@EntityListeners(AuditingEntityListener.class)` for audits. Use `@OneToMany/@OneToOne` with cascade and orphanRemoval.

### Foreign Key to Category Tables (IMPORTANT)

Category tables (`dm_*`) have a `ma` (code) column as their natural identifier. Business tables (`nv_*`) reference these via FK columns that store the `ma` value, NOT the `id`. Examples:
- `nv_chi_tiet_bhxh.ma_co_quan_bhxh` → `dm_co_quan_bhxh.ma`
- `nv_chi_tiet_bhxh.ma_phuong_an_dong` → `dm_phuong_an_dong_bhxh.ma`
- `nv_nguoi_tham_gia.ma_quoc_tich` → `dm_quoc_tich.ma`
- `nv_nguoi_tham_gia.ma_dan_toc` → `dm_dan_toc.ma`
- `nv_nguoi_tham_gia.ma_tinh_tt` → `dm_don_vi_hanh_chinh.ma_dbhc`

**Critical**: Always specify `referencedColumnName = "ma"` (or `"ma_dbhc"` for admin units) in `@JoinColumn`. When querying, JOIN with category tables to fetch the `ten` (name) field for display:
```java
// Entity mapping
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "ma_co_quan_bhxh", referencedColumnName = "ma")
private DmCoQuanBhxh coQuanBhxh;

// Service/DTO mapping - use .getTen() for display
.coQuanBhxh(entity.getCoQuanBhxh() != null ? entity.getCoQuanBhxh().getTen() : null)

// Native query example
SELECT h.ma_ho_so, pa.ten AS ten_phuong_an
FROM nv_ho_so_dang_ky h
LEFT JOIN nv_chi_tiet_bhxh ct ON h.id = ct.ho_so_id
LEFT JOIN dm_phuong_an_dong_bhxh pa ON ct.ma_phuong_an_dong = pa.ma
```

### API Layer

CMS APIs: `/api/v1/**` (requires `@PreAuthorize` permission checks). App APIs: `/api/app/v1/**` (authentication only). Use `@Tag`, `@Operation` for Swagger. Return `ResponseEntity<ApiResponse<T>>`. Example: `@PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.BHXH_XEM_HO_SO)")`.

### Response Wrappers

`ApiResponse<T>` for single items: `{code, message, data}`. `PageResponse<T>` for paginated responses. Use `ResponseUtil.ok()` helper.

### Error Handling

Throw `BusinessException` with `MessageResponseDict`. Global handling in `ExceptionController`. Use Jakarta validation (`@NotNull`, `@Valid`, custom messages). Example: `throw new BusinessException(MessageResponseDict.HO_SO_NOT_FOUND)`.

### Service Layer

Use `@Service`, `@RequiredArgsConstructor`, `@Transactional` for data modification, `@Slf4j` for logging. Use event publishing (`ApplicationEventPublisher`) for audit logging on state changes.

### Repository Layer

Extend `JpaRepository<Entity, ID>`. Use `@Query` with JPQL. Use `nativeQuery = true` for complex SQL with CTEs. Use projection interfaces for list views. Bulk operations: `saveAll()`, batch size 500-600.

### Logging

Use `@Slf4j`. Format: `log.info("Action description: id={}, status={}", id, status)`. Levels: info (business events), debug (details), warn (recoverable), error (exceptions).

### Validation

Use Jakarta validation on DTOs/Requests. Use `@Valid` for nested validation. Custom error messages in `@NotNull(message = "...")`.

### MapStruct

Use MapStruct for Entity ↔ DTO mapping. Define mapper interfaces with `@Mapper(componentModel = "spring")`. Use `@Mapping` for custom field mappings.

### Dependencies

Spring Boot 3.4.4, Lombok, MapStruct, Springdoc OpenAPI 2.8.6, JPA, MySQL 8, JWT 0.12.3, Flyway, Redis 3.4.4, WebFlux, Gson 2.8.9, OpenTelemetry. **Never add new libraries without checking pom.xml first.**

### Configuration

Database in `application.yml`. Security separates APP vs CMS permissions. Flyway migrations in `src/main/resources/db/migration/` (currently disabled). Swagger at `/swagger-ui.html`. Redis TTL: 60 minutes default. HikariCP connection pool: max-size 100, min-idle 10.

### Permission-based Access

CMS: `@PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.PERMISSION_NAME)")`. App: Only `@authenticated()` required. Permission constants in `PermissionConstants` (all 17 permissions defined). Permission checking via `PermissionChecker` bean.

### State Machine Pattern

Use `HoSoStateMachine` for Hồ sơ status transitions. States: `CHO_THU_TIEN` → `DA_THU_TIEN` → `CHO_DUYET_NOI_BO` → `CHO_NOP_CQ_BHXH` → `CQ_BHXH_DANG_XU_LY` → `THANH_CONG`/`BI_TU_CHOI`/`DA_HUY`. Events defined in `HoSoEvent`. Call `HoSoStateMachine.nextTrangThai(currentStatus, event)` to get next state.

### Event-Driven Architecture

Publish `HoSoStatusChangedEvent` on state changes for audit logging. Use `@EventListener` to handle events. Event publishing in service layer.

### Transaction Management

Use `@Transactional` at service method level for write operations. Avoid mixing read/write operations. Batch size: 500 (configured in application.yml). Use `saveAll()` for bulk operations.

### Database Migrations

Flyway migrations in `src/main/resources/db/migration/`. Currently disabled (`spring.flyway.enabled=false`). Use snake_case for table/column names. Use `baseline-on-migrate: true` for existing databases.

### Caching

Redis caching via `RedisFunction` and `KeyConst`. Default TTL: 60 minutes. Use `@Cacheable`, `@CacheEvict` sparingly. Cache keys defined in `KeyConst`.

### External API Integration

Use `WebClient` from WebFlux for external calls. Configure timeouts and max retries in `application.yml`. Use `Gson` for JSON parsing. Example: `BhxhAdapterClientService` for IVAN integration.

### Observability

OpenTelemetry tracing enabled (`management.tracing.sampling.probability=1.0`). Use `@Slf4j` with tracing context propagation. Monitor via Actuator endpoints.

### Important Notes

- Vietnamese domain - use Vietnamese for business logic naming (methods, messages, logs)
- DB: snake_case, Java: camelCase
- State machine for Hồ sơ status (`HoSoStateMachine`) - always validate transitions
- Sự vụ (incident) system for corrections - blocks hồ sơ while unresolved
- No tests - prioritize testing new features
- Bulk operations: batch size 500-600 (configured in application.yml)
- Use projection interfaces for list views to avoid N+1 queries
- Always validate permissions in CMS endpoints via `@PreAuthorize`
- Use `@Transactional` for write operations, read-only for queries
- All external calls use WebClient from WebFlux with proper error handling
- **FK to category tables**: `nv_*.ma_xxx` columns store `dm_xxx.ma` (code), NOT `id`. Always JOIN or use `@ManyToOne` to fetch `.getTen()` for display
