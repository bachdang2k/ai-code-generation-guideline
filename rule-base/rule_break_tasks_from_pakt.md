# Rule: Break Tasks from PAKT into Solution Files

> **Version**: 1.4 (Ultra-Concise + Code-First)
> **Input**: PAKT document | **Output**: `pakt/{feature}/task_{XX}_{name}_solution.md`
> **References**: `rule_technical.md` (coding), `TEST.md` (testing)
> **Critical**: All code must be actual Java, not pseudocode. README.md is MANDATORY.

---

## 📋 Table of Contents

1. [Quick Reference](#quick-reference)
2. [Splitting Rules](#splitting-rules)
3. [Template 00: Data Model](#template-00-data-model)
4. [Template: Business Task](#template-business-task)
5. [Template: Integration Task](#template-integration-task)
6. [Test Auth Setup (All Tests)](#test-auth-setup-all-tests)
7. [Process & Sync](#process--sync)
8. [README Template (MANDATORY)](#readme-template-mandatory)
9. [Quality Checklist](#quality-checklist)
10. [Key Principles](#key-principles)

---

## Quick Reference

**PAKT → Tasks**:
- Task 00: §5 (Data Model), §6 (Cross-Domain), §7 (Error Codes)
- Business: §2 (Endpoints), §4 (APIs), §6 (Logic), §3 (Enrichment), §8 (Security)
- Integration: §7 (External), §3 (Async)

**Task Types**: 00 Foundation (8-12h), Business (8-16h), Integration (8-12h)

**Naming**: `task_{XX}_{name}_solution.md` (XX=01,02,03; name=lowercase_underscores)

---

## Splitting Rules

**Split**: >7 endpoints, different enrichment, external integration, >16h

**Group**: Detail+Update (always), Create+List (if ≤16h), same enrichment source

---

## Template 00: Data Model

```markdown
# Task 00: Data Model - {Feature}

**PAKT**: §5, §6, §7 | **Hours**: 8-12

## 1. Entities (PAKT §5)
**Package**: `vn.vivas.bh.entity`

**Fields**: id (PK, AUTO_INCREMENT), fieldName (column: field_name, nullable: false), trangThai (enum, column: trang_thai), ngayTao/ngayCapNhat (audit)

**Relationships**: @OneToMany (children), @ManyToOne (parent)

**Enrichment** (PAKT §5): ✅Add columns: {list}, ❌Lookup: {list}, ⚡Async: {list}

**Code**: rule_technical.md §4, AGENTS.md (@Table, @Column, @Enumerated, @CreatedDate, @LastModifiedDate)

## 2. Enums
**Package**: `vn.vivas.bh.enums` | **Values**: VALUE_1, VALUE_2

**Code**: rule_technical.md §4.4 (PascalCase, @Getter, @AllArgsConstructor)

## 3. Repositories (PAKT §4)
**Package**: `vn.vivas.bh.repository`

**Methods**: findByFieldName(String): Optional<Entity>, findByTrangThai(TrangThai): List<Entity>

**Custom**: @Query (JPQL), nativeQuery with projection

**DB Logic**: Tables: {list}, Joins: {list}, Fields: {list}, Purpose: {from PAKT}

**Projection**: InterfaceNameProjection (id, field1, field2)

**Code**: rule_technical.md §4.3 (@Query, nativeQuery, projection)

## 4. Error Codes (PAKT §7)
**Add**: ENTITY_NOT_FOUND (100001), ENTITY_FIELD_REQUIRED (100002, "%s")

**Code**: rule_technical.md §4.1 (success=0, errors=100000-400000, HttpStatus=OK, List<String> params)

## 5. State Machine (PAKT §5)
**States**: {list} | **Transitions**: {list}

**Class**: EntityStateMachine.nextTrangThai(TrangThai, Event): TrangThai

**Code**: rule_technical.md §4.3, AGENTS.md (State Machine Pattern)

## 6. Cross-Domain (PAKT §6)
**DanhMucService.getNsdpRate(Integer): BigDecimal** (SLA: <100ms cached, Error: Return 0)

## 7. Testing
**Repository IT**: EntityNameRepositoryIT (@Import(TestcontainersConfiguration.class), CRUD+custom query tests, clean DB @BeforeEach

**Code**: TEST.md §4, §2 (TestContainers, @ServiceConnection), §5 (naming: *IT), §6 (best practices)

## 8. Checklist
- [ ] Entities + relationships
- [ ] Enums
- [ ] Repositories with DB logic
- [ ] Error codes
- [ ] State machine (if applicable)
- [ ] Cross-domain contracts
- [ ] Repository IT tests
- [ ] `./mvnw clean compile` - no errors
```

---

## 3.5 Code Specification Requirements (CRITICAL)

**Rule**: `*_solution.md` files must contain ACTUAL Java code, not pseudocode.

### Entity Example:
```java
package vn.vivas.bh.entity.nghiepvu;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

@Entity
@Table(name = "nv_doi_soat_thu_lao")
@EntityListeners(AuditingEntityListener.class)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class NvDoiSoatThuLao {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    
    @Column(name = "ten_doi_soat", nullable = false, unique = true, length = 200)
    private String tenDoiSoat;
    
    @Column(name = "tu_ngay", nullable = false)
    private LocalDate tuNgay;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "trang_thai", nullable = false)
    private TrangThaiDoiSoat trangThai;
    
    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

### State Machine Example:
```java
package vn.vivas.bh.statemachine;

import vn.vivas.bh.enums.TrangThaiDoiSoat;
import vn.vivas.bh.exception.BusinessException;
import vn.vivas.bh.constant.MessageResponseDict;

public class DoiSoatThuLaoStateMachine {
    
    public static void validateCanModify(TrangThaiDoiSoat trangThai) {
        if (TrangThaiDoiSoat.DA_DOI_SOAT.equals(trangThai)) {
            throw new BusinessException(MessageResponseDict.DOI_SOAT_INVALID_STATE);
        }
    }
}
```

### Repository Example:
```java
package vn.vivas.bh.repository.nghiepvu;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import vn.vivas.bh.entity.nghiepvu.NvDoiSoatThuLao;
import vn.vivas.bh.repository.projection.DoiSoatListProjection;

@Repository
public interface DoiSoatThuLaoRepository extends JpaRepository<NvDoiSoatThuLao, Integer> {
    
    boolean existsByTenDoiSoat(String tenDoiSoat);
    
    @Query(value = """
        SELECT ds.id as id,
               ds.ten_doi_soat as tenDoiSoat,
               a.name as tenDaiLyThu,
               cq.ten as tenCoQuanBhxh,
               COUNT(dshs.id) as soLuongHoSo,
               SUM(dshs.thu_lao_duoc_huong) as tongThuLao
        FROM nv_doi_soat_thu_lao ds
        LEFT JOIN agency a ON a.id = ds.dai_ly_thu_id
        LEFT JOIN dm_co_quan_bhxh cq ON cq.id = ds.co_quan_bhxh_id
        LEFT JOIN nv_doi_soat_thu_lao_ho_so dshs ON dshs.doi_soat_thu_lao_id = ds.id
        WHERE ds.deleted_at IS NULL
          AND (:trangThai IS NULL OR ds.trang_thai = :trangThai)
          AND (:tenDoiSoat IS NULL OR ds.ten_doi_soat LIKE %:tenDoiSoat%)
        GROUP BY ds.id
        ORDER BY ds.created_at DESC
        """, nativeQuery = true)
    Page<DoiSoatListProjection> findAllWithAggregation(
        @Param("trangThai") String trangThai,
        @Param("tenDoiSoat") String tenDoiSoat,
        Pageable pageable
    );
}
```

### Service Method Example:
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class DoiSoatThuLaoService {
    
    private final DoiSoatThuLaoRepository repository;
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public DoiSoatResponse taoDoiSoat(TaoDoiSoatRequest request) {
        // 1. Auth
        AccountInfo user = AuthenticationService.getCurrentUserRequired();
        log.info("[{}/TAO_DOI_SOAT] tenDoiSoat={}, userId={}", request.getTenDoiSoat(), user.getId());
        
        // 2. Validate
        if (repository.existsByTenDoiSoat(request.getTenDoiSoat())) {
            throw new BusinessException(MessageResponseDict.DOI_SOAT_TEN_DUPLICATE);
        }
        
        // 3. Build entity
        NvDoiSoatThuLao entity = NvDoiSoatThuLao.builder()
            .tenDoiSoat(request.getTenDoiSoat())
            .tuNgay(request.getTuNgay())
            .denNgay(request.getDenNgay())
            .daiLyThuId(user.getDaiLyId())
            .trangThai(TrangThaiDoiSoat.DANG_DOI_SOAT)
            .createdByUserId(user.getId())
            .build();
        
        // 4. Persist
        repository.save(entity);
        
        // 5. Event
        eventPublisher.publishEvent(DoiSoatCreatedEvent.builder()
            .doiSoatThuLaoId(entity.getId())
            .build());
        
        // 6. Return
        log.info("[{}/TAO_DOI_SOAT] SUCCESS id={}, userId={}", entity.getId(), user.getId());
        return DoiSoatResponse.builder()
            .id(entity.getId())
            .tenDoiSoat(entity.getTenDoiSoat())
            .trangThai(entity.getTrangThai().getTen())
            .build();
    }
}
```

### Validation Rule:
- [ ] Entity has full annotations (@Entity, @Table, @Column, @Enumerated, etc.)
- [ ] Enum has @Getter @AllArgsConstructor
- [ ] Repository has complete @Query with native SQL
- [ ] State machine has full validation logic
- [ ] Service has complete 9-step implementation pattern
- [ ] No pseudocode allowed

---

## Template: Business Task

```markdown
# Task XX: {API Name} - {Feature}

**PAKT**: §2, §4, §6, §3, §8 | **Hours**: X | **Dependencies**: Task 00 ✅

## 1. API Contract (PAKT §4)
| Method | Path | Request | Response | Errors |
|--------|------|---------|----------|--------|
| POST | /api/app/v1/... | RequestDTO | ResponseDTO | ERROR_1 |

## 2. Request DTO (PAKT §4)
**Class**: RequestDTO | **Package**: `vn.vivas.bh.request`

**Fields**: fieldName (@NotNull, @Size(min=10, max=10)), cccd (@NotNull, @Size(min=12, max=12)), ngaySinh (@Past)

**Code**: rule_technical.md §4.3 (@Data @Builder, Jakarta validation)

## 3. Response DTO (PAKT §4)
**Class**: ResponseDTO | **Package**: `vn.vivas.bh.response`

**Fields**: id, maHoSo, trangThai, hoTen, calculatedField

**Code**: rule_technical.md §4 (@Data @Builder @FieldDefaults(PRIVATE), ApiResponse<T> wrapper)

## 4. Service Method (PAKT §6)
**Class**: {Feature}Service | **Package**: `vn.vivas.bh.service`

**Method**: `@Transactional public ResponseDTO methodName(RequestDTO request)`

**DB Logic** (PAKT §4): Tables: {list}, Joins: {list}, Filters: {list}, Purpose: {from PAKT}

**Steps** (PAKT §6): 1.Auth 2.Validate 3.Enrichment (sync calls) 4.Load (query) 5.Logic 6.Persist 7.Event (if async) 8.Log 9.Return

**Cross-Feature** (PAKT §2, §7): Sync: {services/methods}, Async: {events}

**Code**: rule_technical.md §4.3 (service pattern), §4.2 (logging), §4.1 (errors), §4.5 (ABAC if CMS)

## 5. Controller (PAKT §4, §8)
**Class**: {Feature}Controller | **Package**: `vn.vivas.bh.controller`

**Endpoint**: {GET/POST/PUT/DELETE} /api/{app|portal}/v1/...

**Request**: @Valid @RequestBody RequestDTO | **Response**: ResponseEntity<ApiResponse<ResponseDTO>>

**Security**: App (auth only), Portal (@PreAuthorize permission)

**ABAC** (if CMS): Apply listAgentID filtering

**Code**: rule_technical.md §4.4 (controller), §4.5 (security), AGENTS.md (@Tag, @Operation)

## 6. Mapper
**Interface**: {Feature}Mapper | **Package**: `vn.vivas.bh.mapper`

**Methods**: toResponse(Entity), toEntity(RequestDTO)

**Code**: AGENTS.md (MapStruct, @Mapper(componentModel="spring"))

## 7. Async Enrichment (PAKT §3)
**Event**: {Event}Event | **Listener**: @EventListener @Async on{Event}({Event}Event event)

**Logic**: Background enrichment, error handling (don't throw)

**Code**: rule_technical.md §4.3 (event publishing), AGENTS.md (Event-Driven Architecture)

## 8. Unit Tests (TEST.md §3)
**Class**: {Feature}ServiceTest | **Package**: vn.vivas.bh.service

**Tests**: should{Method}WhenValidInput(), shouldThrowWhen{Invalid()}

**Setup**: @ExtendWith(MockitoExtension.class), @Mock dependencies, @InjectMocks service, Vietnamese names

**Auth**: See "Test Auth Setup" below

**Code**: TEST.md §3 (@ExtendWith, @Mock, @InjectMocks), §5 (naming), §6 (Given-When-Then)

## 9. Integration Tests (TEST.md §4)
**Class**: {Feature}ControllerIT | **Package**: vn.vivas.bh

**Tests**: should{Method}(), shouldReturn{ErrorCode}When{Invalid()}

**Setup**: @SpringBootTest, @AutoConfigureMockMvc, @Import(TestcontainersConfiguration.class), clean DB @BeforeEach

**Auth**: See "Test Auth Setup" below

**Code**: TEST.md §4 (TestContainers), §2 (@ServiceConnection), §6 (best practices), §8 (@MockitoBean)

## 10. Checklist
- [ ] DTOs with validation
- [ ] Service method with DB logic
- [ ] Controller with security
- [ ] ABAC (if CMS)
- [ ] Mapper
- [ ] Event/listener (if async)
- [ ] Unit tests (≥80%)
- [ ] Integration tests
- [ ] `./mvnw test` - pass

## 11. Sync Points
**Depends on**: Task 00 ✅

**Uses**: Entities, Repositories, Enums, Error codes (Task 00)

**Events**: Publishes {Event}Event → Listened by Task YY
```

---

## Template: Integration Task

```markdown
# Task XX: {Integration} - {Feature}

**PAKT**: §7, §3 | **Hours**: 8-12 | **Dependencies**: Task 00 ✅

## 1. Configuration (PAKT §7)
**application.yml**:
```yaml
external:
  {service}:
    base-url: ${SERVICE_URL:http://localhost:8080}
    timeout: 5000
    max-retries: 3
```

**Config Class**: {Service}Config (@ConfigurationProperties, baseUrl/timeout/maxRetries fields)

## 2. WebClient Bean
**Method**: webClient({Service}Config config): WebClient

**Setup**: HttpClient with timeout, responseTimeout

**Return**: WebClient.builder().baseUrl().clientConnector().build()

**Code**: rule_technical.md, AGENTS.md (WebClient from WebFlux)

## 3. Adapter Service (PAKT §7)
**Class**: {Service}AdapterClientService | **Package**: `vn.vivas.bh.service`

**Method**: ReturnType methodName(ParamType param)

**API**: {method} {path} | **Returns**: Type or throws BusinessException

**Error** (PAKT §7): 404 → {ERROR_CODE}, 500+ → {ERROR_CODE}

**Retry**: retryWhen(Retry.backoff(3, Duration.ofSeconds(1))), timeout(Duration.ofSeconds(5))

**Code**: rule_technical.md §4.1, AGENTS.md (WebClient, Gson)

## 4. Event Listener (PAKT §3)
**Event**: {Event}RequestedEvent | **Listener**: @EventListener @Async on{Event}Requested({Event}RequestedEvent event)

**Logic**: Load entity, call external service, update entity, save, log

**Error**: Don't throw (async)

**Code**: rule_technical.md §4.3 (event publishing), AGENTS.md (Event-Driven Architecture)

## 5. Unit Tests (TEST.md §3)
**Class**: {Service}AdapterClientServiceTest

**Tests**: should{Method}WhenSuccess(), shouldThrowWhen{Error()}

**Setup**: @ExtendWith(MockitoExtension.class), Mock WebClient chain, @InjectMocks service

**Code**: TEST.md §3, §6

## 6. Integration Tests (TEST.md §4)
**Class**: {Service}AdapterClientServiceIT

**Setup**: @SpringBootTest, @Import(TestcontainersConfiguration.class), WireMockExtension

**Tests**: should{Method}WhenExternalApiReturns200(), shouldThrowWhenExternalApiReturns404()

**WireMock**:
```java
@RegisterExtension static WireMockExtension wireMock = WireMockExtension.newInstance()
    .options(wiremockConfig().port(8080)).build();

@BeforeEach void setUp() {
    wireMock.stubFor(get(urlPathEqualTo("/api/path"))
        .willReturn(aResponse().withStatus(200).withBody("{}")));
}
```

**Auth**: See "Test Auth Setup" below

**Code**: TEST.md §4, §6

## 7. Checklist
- [ ] application.yml
- [ ] Config class
- [ ] WebClient bean
- [ ] Adapter service
- [ ] Error handling
- [ ] Event listener (if async)
- [ ] Unit tests (WebClient mocking)
- [ ] Integration tests (Wiremock)
- [ ] `./mvnw test` - pass

## 8. Dependencies
**Depends on**: Task 00 ✅

**Used by**: Business tasks (service calls)

**Events**: Listens to business task events
```

---

## Test Auth Setup (All Tests)

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

---

## Process & Sync

**Steps**: Read PAKT → Extract Task 00 → Identify tasks → Apply split/group → Order by dependency → Create files → Document DB logic → **Create README.md (MANDATORY)** → Validate code completeness (no pseudocode) → Final review

**Sync**:
- Shared (Task 00): Entities, Repositories, Enums, Error codes
- Services: `@Autowired private ExternalServiceAdapter externalService;`
- Events: `eventPublisher.publishEvent(Event.builder()...build());`
- Transactions: @Transactional at service method, validate BEFORE, events for async
- **README**: Must include task matrix, dependency graph, sync points, progress tracking

---

## README Template (MANDATORY)

```markdown
# {Feature} - Task Solutions

**PAKT**: {feature}_pakt.md  
**Created**: {date}  
**Status**: In Progress

## Overview
{Brief description of feature}

## Task Dependency Matrix

| Task | ID | Name | Hours | Dependencies | Output | Status |
|------|----|----|-------|--------------|--------|--------|
| Task 00 | 00 | Data Model | 8-12 | None | Entities, Repos, Enums | ⏳ |
| Task 01 | 01 | Create API | 8-12 | Task 00 ✅ | Service, Controller | ⏳ |
| Task 02 | 02 | List API | 8-12 | Task 00 ✅ | Service, Controller | ⏳ |

## Dependency Graph
```
Task 00 (Foundation)
    ├─→ Task 01 (Create API)
    └─→ Task 02 (List API)
         └─→ Task 03 (Export Excel)
```

## Sync Points

### Shared (Task 00)
- Entities: `EntityName1`, `EntityName2`
- Repositories: `EntityName1Repository`, `EntityName2Repository`
- Enums: `EnumName`
- Error codes: XXXxx range

### Events
- Task 01 → Task 02: Publishes `{Event}Event`

### Services
- Task 03 uses: `{ServiceName}.{methodName}(params)`

## Progress
- [ ] Task 00: Data Model (0/8 sections)
- [ ] Task 01: Create API (0/10 sections)
- [ ] Task 02: List API (0/10 sections)

## Notes
- FK to category tables: use `referencedColumnName = "ma"`
- Display names: use `.getTen()` from @ManyToOne relationships
- All code must be actual Java, not pseudocode
```

---

## Quality Checklist

- [ ] All 10 PAKT sections mapped
- [ ] Task 00: complete data model
- [ ] Business tasks: service methods with DB logic
- [ ] Integration tasks: WebClient config
- [ ] References: rule_technical.md, TEST.md
- [ ] DB logic: tables/joins/fields/purpose
- [ ] Test auth: fake UserDetail
- [ ] Files: `pakt/{feature}/`
- [ ] Names: `task_{XX}_{name}_solution.md`
- [ ] **README.md created (MANDATORY)**
- [ ] **All code is actual Java, not pseudocode**
- [ ] **Entities have complete annotations**
- [ ] **Repositories have complete @Query implementations**
- [ ] **State machines have full validation code**
- [ ] **Services follow 9-step pattern with actual code**
- [ ] Each task ≤16 hours
- [ ] Ready for code generation without interpretation

---

## Key Principles

⭐ **PAKT = WHAT**, **Task Solutions = HOW with ACTUAL CODE**

⭐ **References**: rule_technical.md (coding), TEST.md (testing), AGENTS.md (conventions)

⭐ **DB Logic**: Tables, Joins, Filters, Purpose (from PAKT §4)

⭐ **Test Auth**: Fake UserDetail (Role.ADMIN, ALL_PERMISSIONS, daiLyId=null) - module doesn't manage Auth

⭐ **Task 00** = Complete data model with FULL Java code (no pseudocode)

⭐ **Business** = DTOs + Service + Controller + Tests (with actual implementations)

⭐ **Integration** = WebClient + Events + Tests (with actual implementations)

⭐ **README** = MANDATORY task index and dependency tracking (not optional)

⭐ **NO PSEUDOCODE** = All code must be compilable Java (Entity, Repository, Service, Controller, etc.)

⭐ **Output** = Ready for code generation without interpretation

⭐ **Code Examples** = Use §3.5 Code Specification Requirements as reference

---

**End of Rule v1.4 (Ultra-Concise + Code-First)**
