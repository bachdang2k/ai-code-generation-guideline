# Rule: Generate PAKT (Technical Solution Scope) from Acceptance Criteria Checklist

> **Version**: 1.3 (Concise)
> **Purpose**: Generate technical scope from acceptance criteria
> **Input**: v5.0 Acceptance Criteria Checklist
> **Output**: PAKT Document (WHAT to code, not HOW) `pakt/{feature}/{feature_name}_pakt.md`
> **Flow**: URD → Checklist → PAKT → Task Files → Code

---

## Quick Reference

**PAKT defines**: API contracts, data model, business logic scope, data enrichment patterns, integrations, security, impact

**PAKT does NOT define**: Implementation details, full code examples (saved for Task Files)

**Format**: Pseudocode only (DTOs: field names/types; Logic: flow descriptions; Queries: tables/joins/fields)

---

## 📋 Table of Contents

1. [Quick Start Guide](#quick-start-guide)
2. [Quick Reference](#quick-reference)
3. [10-Section Structure](#10-section-structure)
4. [§1 Overview & Scope](#1-overview--scope)
5. [§2 Acceptance Criteria Mapping](#2-acceptance-criteria-mapping)
6. [§3 Architecture & Data Flow](#3-architecture--data-flow)
7. [§4 API Design](#4-api-design)
8. [§5 Data Model & State](#5-data-model--state)
9. [§6 Business Logic Scope](#6-business-logic-scope)
10. [§7 Integration & Enrichment Scope](#7-integration--enrichment-scope)
11. [§8 Security Scope](#8-security-scope)
12. [§9 Impact & Dependencies](#9-impact--dependencies)
13. [§10 Implementation Scope](#10-implementation-scope)
14. [Quality Gates](#quality-gates)
15. [Key Principles](#key-principles)

---

## Quick Start Guide

### What to Do Before Asking Me to Code

**Step 1**: Read and understand this rule (5-10 minutes)
- Understand the 10-section structure
- Know which sections are mandatory (Sections 1-10)
- Recognize that PAKT defines WHAT to code, not HOW

**Step 2**: Review your acceptance criteria checklist
- Identify user stories, scenarios, validations, business rules
- Extract API contracts, data model requirements, business logic

**Step 3**: Generate the PAKT using the templates
- Copy Section 1 template (Overview & Scope)
- Copy Section 2 template (Mapping)
- Copy Section 3 template (Architecture)
- Copy Section 4 template (API Design)
- Copy Section 5 template (Data Model)
- Copy Section 6 template (Business Logic)
- Copy Section 7 template (Integration)
- Copy Section 8 template (Security)
- Copy Section 9 template (Impact)
- Copy Section 10 template (Implementation)

**Step 4**: Validate the PAKT before showing it to developers
- Run through the Quality Gates checklist (Section 14)
- Ensure all 10 sections are populated
- Verify PAKT defines WHAT, not HOW

**Step 5**: Get approval and proceed to task breaking

---

## Quick Reference

| # | Section | Purpose |
|---|---------|---------|
| 1 | Overview & Scope | Feature summary, enrichment context, scope in/out |
| 2 | Acceptance Criteria Mapping | Scenarios → endpoints → enrichment → logic |
| 3 | Architecture & Data Flow | Request flow, state machine, enrichment patterns |
| 4 | API Design | Endpoints, DTOs (pseudocode), validations |
| 5 | Data Model & State | Entities, tables, relationships, enrichment mapping |
| 6 | Business Logic Scope | Calculations, validations, rules (pseudocode) |
| 7 | Integration & Enrichment | External APIs, internal cross-feature enrichment |
| 8 | Security Scope | Authentication, authorization, data scoping |
| 9 | Impact & Dependencies | DDL, performance, compatibility |
| 10 | Implementation Scope | Tasks, enrichment checklist, DoD |

---

## Section Templates

### §1 Overview & Scope

```markdown
## 1. Overview & Scope

**Feature Summary**: [Name] | Type: [New/Enhancement] | Module: [App/Portal] | Complexity: [Simple/Medium/Complex]

**Business Context**
- User Stories: US-01, US-02
- Actors: [Agent, Admin]
- Business Value: [Why]

**Technical Context**
- Service Class: [FeatureName]Service
- Primary Entities: [Entity1, Entity2]
- State Machine: [Yes/No]

**Data Enrichment Context**
- Required External Data: [BHXH API, etc.]
- Required Internal Data: [Other modules/entities]
- Enrichment Strategy: [Synchronous API / Async Event / Hybrid]
- Calculation Ownership: [Domain responsible]

**Scope**
In Scope: [List]
Out of Scope: [List]

**URD References**: Primary: [path], Related: [paths]
```

---

### §2 Acceptance Criteria Mapping

```markdown
## 2. Acceptance Criteria Mapping

### Scenario → Endpoint Mapping
| Scenario | API Endpoint | Purpose |
|----------|-------------|---------|
| US-01, S1.1 | POST /api/app/v1/ho-so | Create hồ sơ |
| US-01, S1.2 | GET /api/app/v1/bhxx/lookup/{maBHXH} | Lookup participant |

### Scenario → Validation Mapping
| Scenario | Validation Rule | Implementation |
|----------|----------------|----------------|
| Create profile | CCCD: Required, 12 digits | @NotNull, @Size(min=12, max=12) |
| Create profile | Ngày sinh: Not future, age >= 15 | @Past, @Custom validator |

### Scenario → Data Enrichment Mapping
| Scenario | Data Source | Enrichment Type | Sync/Async | Implementation |
|----------|-------------|-----------------|------------|----------------|
| US-01, S1.2 | BHXH API | Participant info | Sync (user-triggered) | BhxhAdapterClientService |
| US-02 | Province catalog | NSDP rate | Sync (cached) | DanhMucCache |
| US-03 | HoSo history | Payment totals | Async (background) | @EventListener(PaymentEvent) |

### Scenario → Business Logic Mapping
| Scenario | Business Rule | Implementation Scope |
|----------|--------------|---------------------|
| Create profile | Method: existing vs new | if (participant.hasBhxh()) DONG_TIEP else DANG_KY_MOI |
| Calculate payment | Monthly formula | months × amount × 22% - NSNN - NSĐP |
| State change | CHỜ THU_TIEN → ĐÃ THU_TIEN | HoSoStateMachine.nextTrangThai(current, event) |
```

---

### §3 Architecture & Data Flow

```markdown
## 3. Architecture & Data Flow

### System Architecture
[Client] → [Controller] → [Service: {Feature}Service] → [Repository] → [Database: MySQL 8]

### Request Flow
1. Controller.{method}() receives RequestDTO
2. Service.{method}() validates input
3. Service loads entities (Repository)
4. Service applies business logic
5. Service persists entities
6. Service publishes event (if state change)
7. Service returns ResponseDTO

### State Machine (if applicable)
States: CHO_THU_TIEN, DA_THU_TIEN, CHO_DUYET_NOI_BO, CHO_NOP_CQ_BHXH, CQ_BHXH_DANG_XU_LY, THANH_CONG, BI_TU_CHOI, DA_HUY
Transitions: CHO_THU_TIEN → (payment) → DA_THU_TIEN → (submit) → CHO_DUYET_NOI_BO → (approve) → CHO_NOP_CQ_BHXH → (submit) → CQ_BHXH_DANG_XU_LY → (success) → THANH_CONG
Implementation: HoSoStateMachine.nextTrangThai(current, event)
Hard Rules: Profiles with active incidents cannot transition; Every state change publishes HoSoStatusChangedEvent

### Data Enrichment Patterns
**Synchronous** (user-facing): Service enriches data → applies logic → persists → returns
**Asynchronous** (background): Service creates entity → publishes event → listener enriches → updates
**Hybrid** (immediate + background): Service returns with partial data → async completes → notification

| Factor | Synchronous | Asynchronous | Hybrid |
|--------|-------------|--------------|--------|
| User waiting? | Yes | No | Yes (partial) |
| Data available? | Immediate | Delayed/Heavy | Partial immediate |
| Example | BHXH lookup | History aggregation | Payment summary |
```

---

### §4 API Design

```markdown
## 4. API Design

### API Endpoints
| Method | Path | Purpose | Request | Response | Ref |
|--------|------|---------|---------|----------|-----|
| POST | /api/app/v1/ho-so | Create hồ sơ | CreateHoSoRequest | HoSoResponse | US-01 |
| GET | /api/app/v1/ho-so/{id} | Get detail | - | HoSoResponse | US-HS02 |

### Request DTOs (Pseudocode)
**CreateHoSoRequest**:
- maBhxh: String (required, 10 digits)
- cccd: String (required, 12 digits)
- ngaySinh: LocalDate (required, not future, age >= 15)
- soDienThoai: String (phone pattern)
- coQuanBhxhId: Integer (required)
- phuongThucDong: PhuongThucDong (enum, required)
- soThang: Integer (required, 1-60)
- mucDong: BigDecimal (required, min=1500000)

### Response DTOs (Pseudocode)
**HoSoResponse**:
- id: Integer
- maHoSo: String
- trangThai: TrangThaiHoSo
- hoTen: String
- cccd: String
- ngaySinh: LocalDate
- soTienPhaiDong: BigDecimal
- ngayTao: LocalDateTime

### Validation Matrix
| Field | Validation | Error Code | Message |
|-------|------------|------------|---------|
| maBhxh | Required, 10 digits | MA_BHXH_INVALID | "Mã số BHXH không hợp lệ" |
| cccd | Required, 12 digits | CCCD_INVALID | "Số CCCD không hợp lệ" |

### Query Logic (for JOIN queries)
#### Query: GetHoSoListWithEnrichment
**Purpose**: Retrieve hồ sơ with province names and payment totals
**Required Fields**: hoSoId, maHoSo, trangThai, tenTinh, tongDaDong
**Tables Used**:
- nv_ho_so_dang_ky (HoSo): id, maHoSo, trangThai
- nv_danh_muc_tinh (Tinh): tenTinh
- nv_lich_su_thanh_toan (LichSu): payment amounts
**Join Conditions**: nv_ho_so_dang_ky.ma_tinh = nv_danh_muc_tinh.ma_tinh; nv_ho_so_dang_ky.id = nv_lich_su_thanh_toan.ho_so_id
**Query Scope**: WHERE trang_thai IN ('CHO_THU_TIEN', 'DA_THU_TIEN') GROUP BY id
**Implementation**: Native query with projection interface
```

---

### §5 Data Model & State

```markdown
## 5. Data Model & State

### Entity → Table Mapping
| Entity | Table | PK | Key Fields | Relationships |
|--------|-------|-----|------------|---------------|
| NvHoSoDangKy | nv_ho_so_dang_ky | id | maHoSo, trangThai | @OneToMany ChiTietBhxh |
| NvChiTietBhxh | nv_chi_tiet_bhxh | id | hoSoId, mucDong | @ManyToOne HoSo |

### Entity Definitions (Scope)
**NvHoSoDangKy** (Hồ sơ):
- Fields: id, maHoSo (unique), trangThai, loaiNghiepVu, phuongThuc, mucDong, soThang, soTienPhaiDong, ngayTao, ngayCapNhat
- Relationships: @OneToMany ChiTietBhxh, @OneToMany NguoiThamGia
- Annotations: @Entity, @Table, @Enumerated(EnumType.STRING), @CreatedDate, @LastModifiedDate

### Cross-Entity Enrichment Mapping
| Primary Entity | Enriched From | Relationship | Fields Added | Timing |
|----------------|---------------|--------------|-------------|--------|
| NvHoSoDangKy | NvDanhMucTinh | @ManyToOne (lazy) | tenTinh, maTinh | Query-time |
| NvHoSoDangKy | NvLichSuThanhToan | @OneToMany | tongDaDong | Async |
| NvHoSoDangKy | BHXH API | (non-persistent) | hoTen, cccd | Sync (user-triggered) |

### Enrichment Field Guidelines
✅ Add column (persistent snapshot): Frequently accessed, audit trail, calculations (soTienPhaiDong)
❌ Lookup at query (reference data): Rarely displayed, shared across entities (tenTinh)
⚡ Async populate: Heavy computations, external API dependent (tongDaDong from lich_su)

**DDL Pattern for Money Fields**:
```sql
-- ✅ CORRECT: Persistent snapshot (audit trail needed)
ALTER TABLE nv_ho_so_dang_ky
ADD COLUMN so_tien_phai_dong DECIMAL(19,2) DEFAULT 0.00,
ADD INDEX idx_so_tien (so_tien_phai_dong);

-- ❌ AVOID: Redundant storage (query from nv_lich_su_thanh_toan instead)
```

### State Transitions (if applicable)
| Current | Event | Next | Validation |
|---------|-------|------|------------|
| CHO_THU_TIEN | Payment confirmed | DA_THU_TIEN | Payment received |
| DA_THU_TIEN | Submit | CHO_DUYET_NOI_BO | Agent submits |
| CHO_DUYET_NOI_BO | Approve | CHO_NOP_CQ_BHXH | Admin approves |
Implementation: HoSoStateMachine.nextTrangThai(current, event)
```

---

### §6 Business Logic Scope

```markdown
## 6. Business Logic Scope

### Service Method Structure (Pseudocode)
METHOD taoHoSo(CreateHoSoRequest) RETURNS HoSoResponse
1. Authenticate user
2. Validate input fields
3. Lookup participant from external API
4. Determine registration method
5. Calculate payment amount
6. Create entity with enriched data
7. Persist to database
8. Log business event
9. Return response DTO

### Business Logic (Pseudocode)
#### Method Determination
IF participant.maBhxx exists AND valid THEN
    RETURN PhuongThuc.DONG_TIEP
ELSE
    RETURN PhuongThuc.DANG_KY_MOI
END IF

#### Payment Calculation
IF phuongThuc == MULTI_YEAR THEN
    principal = mucDong * soThang
    discountRate = getDiscountRate(soThang)
    soTien = calculatePresentValue(principal, discountRate, soThang)
ELSE
    soTien = soThang * mucDong * 0.22
    soTien = soTien - calculateNSNN(mucDong, tiLeNSNN)
    soTien = soTien - calculateNSDP(mucDong, tiLeNSDP)
END IF
RETURN soTien.setScale(0, HALF_UP)

#### State Transition Logic
hoSo = repository.findById(hoSoId)
newState = hoSoStateMachine.nextTrangThai(hoSo.trangThai, event)
hoSo.trangThai = newState
hoSo = repository.save(hoSo)
eventPublisher.publishEvent(HoSoStatusChangedEvent)

### Domain-Specific Calculations
**Ownership Pattern**:
IF calculation involves ONLY {feature} entities THEN
    Calculation belongs in {Feature}Service
ELSE IF calculation involves CROSS-DOMAIN entities THEN
    Define {CalculationDomain}Service with clear contract
    Document cross-domain dependencies in PAKT §7
END IF

**Example - Payment Enrichment**:
baseAmount = mucDong * soThang * 0.22
tinh = danhMucService.findByMaTinh(maTinh)  // Cross-domain lookup
nsdpRate = tinh.tiLeNsdpHienHanh  // Owned by DanhMucDomain
nsdpAmount = calculateNsdp(mucDong, nsdpRate)
nsnnAmount = nshnService.calculateNshn(mucDong)  // Owned by NshnDomain
soTien = baseAmount - nsdpAmount - nsnnAmount
RETURN soTien.setScale(0, HALF_UP)

**Cross-Domain Contract**: Service method: DanhMucService.getNsdpRate(Integer maTinh); Dependencies: NvDanhMucTinh; SLA: < 100ms (cached); Error: Return 0 if not found

### Validation Logic
Field Validations: CCCD (12 digits), Ngày sinh (not future, age >= 15), Số điện thoại (phone pattern), Mức đóng (min/max)
Cross-Field Validations: Method + Số tháng (if multi-year, divisible by 12), Payment amount (min/max), State transitions (valid only)

### Error Handling
| Error | Code | Exception |
|-------|------|-----------|
| Invalid CCCD | CCCD_INVALID | BusinessException |
| Hồ sơ not found | HO_SO_NOT_FOUND | BusinessException |
| Invalid state | INVALID_STATE | BusinessException |
```

---

### §7 Integration & Enrichment Scope

```markdown
## 7. Integration & Enrichment Scope

### External System Integrations
#### BHXH Lookup Service
**Capability**: Validate and retrieve participant info by Mã số BHXH
**Trigger**: Agent clicks "Tra cứu" button
**Technical Scope**:
- Service: BhxhAdapterClientService
- Method: ParticipantInfo lookupParticipant(String maBhxx)
- API: GET /api/v1/bhxh/participants/{maBhxx}
- Returns: ParticipantInfo or throws BusinessException
- Error Handling: 404 → MA_BHXH_NOT_FOUND; 500+ → BHXH_API_ERROR
- Configuration: Base URL (from application.yml), Timeout: 5s, Max retries: 3

#### BHXH Authority Submission
**Capability**: Submit declaration forms (D05-TS, D03-TS, TK01-TS)
**Trigger**: Admin completes "Ký số và nộp" workflow
**Technical Scope**:
- Service: BhxhAuthorityService
- Method: CompletableFuture<SubmissionResult> submitDotKeKhai(DotKeKhai)
- Processing: Async (background job)
- SQL Logic: Tables - nv_ho_so_dang_ky, nv_chi_tiet_bhxh, nv_nguoi_tham_gia; Joins - nv_ho_so_dang_ky.id = nv_chi_tiet_bhxh.ho_so_id, nv_ho_so_dang_ky.id = nv_nguoi_tham_gia.ho_so_id; Purpose - Generate D05-TS form

### Internal Cross-Feature Enrichment
**Purpose**: Document data enrichment from other internal modules

| Enrichment Source | Service/Entity | Method | Sync/Async | Error Handling | Data Freshness |
|-------------------|----------------|--------|------------|----------------|----------------|
| Province data | DanhMucService | getTinhInfo(maTinh) | Sync (cached) | Return default | Startup refresh |
| Payment history | LichSuThanhToanService | getTongDaDong(hoSoId) | Async | Partial OK | Real-time |
| Participant status | HoSoService | getTrangThai(hoSoId) | Sync | Cached | 5min TTL |

**Sync Enrichment Pattern** (pseudocode):
METHOD getHoSoDetail(id) RETURNS HoSoResponse
1. Load entity by id
2. Map to response DTO
3. Enrichment: Call danhMucService.getTenTinh(maTinh) - SYNC, cached
4. Enrichment: Call lichSuService.getTongDaDong(id) - pre-computed
5. Return enriched response

**Async Enrichment Pattern** (pseudocode):
METHOD taoHoSo(CreateHoSoRequest) RETURNS HoSoResponse
1. Create entity with minimal data
2. Persist to database
3. Publish HoSoCreatedEvent (hoSoId, cccd)
4. Return response immediately

EVENT LISTENER onHoSoCreated(HoSoCreatedEvent):
1. Extract hoSoId and cccd from event
2. Call bhxhService.lookupParticipant(cccd) - external API
3. Update entity with participant info
4. Save updated entity
```

---

### §8 Security Scope

```markdown
## 8. Security Scope

### Authentication
All APIs: JWT token required
Method: AuthenticationService.getCurrentUserRequired()

### Authorization Matrix
| API | Permission | Role | Data Scope |
|-----|-----------|------|------------|
| POST /api/app/v1/ho-so | Authenticated only | Agent | Own data |
| GET /api/v1/ho-so | BHXH_XEM_HO_SO | Admin | Agency scope |
| POST /api/v1/ho-so/approve | BHXH_DUYET_HO_SO | Admin | Agency scope |

### Data Scoping (ABAC)
**Pattern**:
- Admin (no filter): See ALL data
- Admin (with filter): See filterId + children
- Agency user: See own agency + children
- Agent: See own data only

**Implementation** (pseudocode):
METHOD getAgentIdsForUser(user, daiLyThuId) RETURNS List<Integer>
IF user.isSuperAdmin() THEN RETURN all agent IDs
ELSE IF user.hasDaiLyThuFilter() THEN RETURN daiLyThuId + child agency IDs
ELSE RETURN user's own agent ID only
END IF

### Security Checklist
- [ ] App APIs require JWT
- [ ] Portal APIs have @PreAuthorize
- [ ] Data scoping applied to queries
- [ ] Sensitive data masked in logs
```

---

### §9 Impact & Dependencies

```markdown
## 9. Impact & Dependencies

### Performance Requirements
| Requirement | Metric | Implementation |
|-------------|--------|----------------|
| API response | < 2s | Optimize queries, pagination |
| List query | < 1s | Index on filter fields |
| Concurrent users | 100+ | Connection pool (max 100) |

### DDL Proposals
⚠️ Requires DB team review and all table, column need add comment 

**New Table**: nv_su_vu
```sql
CREATE TABLE nv_su_vu (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ma_su_vu VARCHAR(50) UNIQUE NOT NULL,
    ho_so_id INT NOT NULL,
    loai_su_vu VARCHAR(50) NOT NULL,
    trang_thai VARCHAR(50) NOT NULL,
    so_tien_thay_doi DECIMAL(19,2) NOT NULL,
    FOREIGN KEY (ho_so_id) REFERENCES nv_ho_so_dang_ky(id),
    INDEX idx_ma_su_vu (ma_su_vu),
    INDEX idx_ho_so_id (ho_so_id)
);
```

**New Columns**:
```sql
ALTER TABLE nv_ho_so_dang_ky
ADD COLUMN co_su_vu_dang_xu_ly BOOLEAN DEFAULT FALSE,
ADD INDEX idx_co_su_vu (co_su_vu_dang_xu_ly);
```

### Money Field Addition Pattern
**When adding money/amount fields**:
1. Data Type: DECIMAL(19,2)
2. Default Value: 0.00 (required) or NULL (optional)
3. Indexing: Add index if used in filters/sorts
4. Audit: Consider @CreatedDate if snapshot
5. Enrichment Source: Document if populated from elsewhere

**Cross-Table Enrichment**:
| Target Table | Source Table | Join Key | Enrichment Fields | Add Column? |
|--------------|--------------|----------|-------------------|-------------|
| nv_ho_so_dang_ky | nv_danh_muc_tinh | ma_tinh | ten_tinh, nsdp_rate | No (lookup) |
| nv_ho_so_dang_ky | nv_lich_su_thanh_toan | ho_so_id | tong_da_dong | No (aggregate) |
| nv_ho_so_dang_ky | (calculation) | N/A | so_tien_phai_dong | Yes (snapshot) |

**Decision Tree**:
```
Need to display money data from another process?
├─ Point-in-time snapshot (audit needed)? → Add column (DECIMAL(19,2))
├─ For filtering/sorting? → Add column + index
├─ Computed real-time from multiple rows? → Query at runtime
└─ Reference data rarely displayed? → Lookup at query
```

### Compatibility
| Aspect | Impact | Mitigation |
|--------|--------|------------|
| Existing APIs | No breaking changes | Backward compatible |
| State machine | New states | Update HoSoStateMachine |
| Permissions | New permissions | Update PermissionConstants |

### Risks
| Risk | Mitigation |
|------|------------|
| BHXH API downtime | Retry 3x, manual input fallback |
| State errors | HoSoStateMachine validation |
| Concurrent updates | Optimistic locking, version field |
```

---

### §10 Implementation Scope

```markdown
## 10. Implementation Scope

### Code Patterns Reference
**rule_technical.md**: 4.1 Exception Handling, 4.2 Logging, 4.3 Service Layer, 4.4 API Response, 4.5 Coding Conventions, 4.6 Security
**TEST.md**: 8 Unit Tests, 9 Integration Tests, 10 Naming Conventions

### Task Breakdown Checklist
**Foundation Tasks (Data Model)**
- [ ] Entities: {Entity1}, {Entity2}
- [ ] Repositories: {Repo1}, {Repo2}
- [ ] Enums: {Enum1}, {Enum2}
- [ ] Error codes: {CODE_1}, {CODE_2}
- [ ] Update state machine

**API Tasks (Controllers & DTOs)**
- [ ] DTOs: {Request1}, {Response1}
- [ ] Controller: {FeatureController}
- [ ] Endpoints: POST /api/..., GET /api/.../{id}, GET /api/..., PUT /api/.../{id}

**Service Tasks (Business Logic)**
- [ ] Service class: {Feature}Service
- [ ] Methods: {method1}(), {method2}()
- [ ] Validations: validate{Method}Input()
- [ ] Calculations: tinh{Calculation}()
- [ ] State transitions: {transition}()
- [ ] Events: publish{Event}()
- [ ] Logging: log.info(), log.error()

**Integration Tasks**
- [ ] WebClient config for {ExternalSystem}
- [ ] Service: {ExternalSystemService}
- [ ] Retry logic, error handling

**Testing Tasks**
- [ ] Unit tests: {Feature}ServiceTest (≥80% coverage)
- [ ] Integration tests: {Feature}RepositoryIT (TestContainers)
- [ ] API tests: {Feature}ControllerTest

### Data Enrichment Checklist
**Recognize Enrichment Scenarios**
- [ ] Does this feature need data from other business processes?
- [ ] Does this feature enrich other processes' data?
- [ ] Are there calculations involving cross-domain data?
- [ ] Is audit/history logging required for state changes?
- [ ] Are money fields needed from other tables?

**Document Enrichment in PAKT**
- [ ] §1: Data Enrichment Context populated
- [ ] §2: Scenario → Enrichment Mapping complete
- [ ] §3: Enrichment pattern specified (sync/async/hybrid)
- [ ] §5: Cross-entity mapping documented
- [ ] §5: Field persistence decisions made (add/lookup/async)
- [ ] §6: Domain ownership for calculations defined
- [ ] §7: Internal enrichment services documented
- [ ] §9: DDL proposals for new columns justified

**Implementation**
- [ ] Sync enrichment: Service method calls (cached)
- [ ] Async enrichment: Event listeners defined
- [ ] Hybrid: Partial response + background completion
- [ ] Money fields: DECIMAL(19,2) with index
- [ ] Cross-domain: Service contracts documented

### Definition of Done
**✅ MANDATORY (100%)**
- [ ] All happy paths (§2) working
- [ ] All common use scenarios (§2) working
- [ ] Core validations (§2) enforced
- [ ] Business rules (§2) verified
- [ ] State transitions (§5) validated
- [ ] Permissions (§8) checked
- [ ] Integrations (§7) working
- [ ] Code patterns followed (rule_technical.md)
- [ ] Unit tests ≥80% coverage
- [ ] All error codes in MessageResponseDict

**⚙️ ENHANCED (Optional)**
- [ ] Error scenarios documented
- [ ] Error messages user-friendly

**🎯 BONUS (Optional)**
- [ ] Edge cases tested
```

---

## Quality Gates {"#quality-gates"}

### Completeness
- [ ] All 10 sections populated
- [ ] All scenarios mapped (§2→4, §3→6, §4→4/6, §5→5, §8→8, §9→7)
- [ ] API contracts complete (endpoints, DTOs, validations)
- [ ] Data model complete (entities, tables, state)
- [ ] Business logic complete (pseudocode, calculations)
- [ ] Task breakdown checklist complete

### Technical Quality
- [ ] DTOs defined as pseudocode (name, fields, types only)
- [ ] No Java annotations in PAKT (saved for Task Files)
- [ ] SQL defines tables/fields/joins (no full queries)
- [ ] Business logic as pseudocode (not implementation)
- [ ] Query logic defines tables, joins, required fields
- [ ] References to technical docs included
- [ ] DDL proposals reviewed

### Traceability
- [ ] Every checklist item mapped to PAKT section
- [ ] No orphaned requirements

---

## Key Principles {"#key-principles"}

⭐ **PAKT = Scope Definition**: WHAT to code, not HOW

⭐ **Pseudocode Only**:
- DTOs: Object name, fields, types, validations
- Logic: Flow descriptions (METHOD → RETURNS → steps)
- Queries: Tables, joins, required fields, scope
- No Java annotations or implementation code

⭐ **Enrichment Recognition**: Identify sync/async/hybrid data needs early

⭐ **Traceability**: Every business requirement mapped to technical section

⭐ **Output**: Single PAKT document → Task Files (implementation details)

---

**End of Rule v1.3 (Concise)**
