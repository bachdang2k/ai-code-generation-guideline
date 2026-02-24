# Rule: Break Tasks from Checklist QC

> **Version**: 3.0 (Concise)
> **Purpose**: Convert acceptance criteria checklist into implementation tasks
> **Input**: Checklist QC (from `rule_create_checklist.md`)
> **Output**: Task files in `docs/tasks/{feature_name}/`
> **Workflow**: URD → Checklist QC → Tasks → PAKT → Code

---

## 📋 Table of Contents

1. [Task Structure](#1-task-structure)
2. [Splitting Rules (IF-THEN)](#2-splitting-rules-if-then)
3. [Grouping Rules](#3-grouping-rules)
4. [Checklist Section → Task Mapping](#4-checklist-section--task-mapping)
5. [File Structure & Naming](#5-file-structure--naming)
6. [Task 00 Template (Foundation)](#6-task-00-template-foundation)
7. [Business Task Template](#7-business-task-template)
8. [Step-by-Step Process](#8-step-by-step-process)
9. [Quality Criteria](#9-quality-criteria)
10. [Common Patterns](#10-common-patterns)
11. [Anti-Patterns (Avoid)](#11-anti-patterns-avoid)
12. [Quick Reference](#12-quick-reference)
13. [Checklist Validation](#13-checklist-validation)

---

## 1. Task Structure

### 1.1 What is a Task?

| Aspect | Description |
|--------|-------------|
| **Scope** | One vertical slice: API + DTOs + Service + Validation |
| **Size** | 8-16 hours ideally |
| **Owner** | Single developer |
| **Output** | Complete, testable feature |

### 1.2 Task 00 vs Business Tasks

| Aspect | Task 00 (Foundation) | Business Tasks (01+) |
|--------|---------------------|---------------------|
| **Entities** | FULL entities (all database fields) | Uses Task 00 entities |
| **Repositories** | Complete interfaces (CRUD + queries) | Calls repository methods |
| **DTOs** | None | Creates Request/Response DTOs |
| **Enums** | All domain enums | Uses Task 00 enums |
| **Error Codes** | All error codes | Throws Task 00 error codes |
| **Service** | None | Implements with pseudocode |
| **Controller** | None | Implements endpoints |

---

## 2. Splitting Rules (IF-THEN)

| Rule | Condition | Action |
|------|-----------|--------|
| **2.1** | Section has >7 items | Split by logical grouping |
| **2.2** | Multiple features (different API prefixes) | Split into parallel tracks |
| **2.3** | Complex item (>3 rules/steps) | Separate task |
| **2.4** | Different skill (POI, WebClient, Events) | Separate by skill boundary |
| **2.5** | >16 hours estimate | Consider splitting |

**Examples**:
- 10 endpoints → Split: Create+List, Detail+Update, Delete+Export
- 2 user stories → 2 tracks: Track A (US-A), Track B (US-B)
- Export feature → Separate task (POI library)

---

## 3. Grouping Rules

| Rule | When to Group | Example |
|------|---------------|---------|
| **3.1** | Same checklist section | Section 2 items 2.1-2.3 → One task |
| **3.2** | Detail + Update APIs | Always group (tightly coupled) |
| **3.3** | Forms vertical slice | API + DTOs + Service + Validation |
| **3.4** | Combined ≤16 hours | Group, else separate |

---

## 4. Checklist Section → Task Mapping

| Checklist Section | Maps To Task |
|-------------------|--------------|
| **Section 1**: Feature Overview | All tasks (context) |
| **Section 2**: API Contracts | Task API definition |
| **Section 3**: Data Structures | Business tasks: DTOs (mapped from entities) |
| **Section 4**: Validation | Task validation implementation |
| **Section 5**: Business Logic | Task service layer |
| **Section 6**: Data & State | **Task 00**: FULL entities, repos, state machine |
| **Section 7**: Error Conditions | **Task 00**: Error code definitions |
| **Section 8**: Integration | Separate integration tasks |
| **Section 9**: NFR | Task acceptance criteria |
| **Section 10**: Traceability | Task references section |

---

## 5. File Structure & Naming

### 5.1 Directory Structure

```
docs/tasks/{feature_name}/
├── task_00_shared_data_model.md
├── task_01_create_api.md
├── task_02_list_api.md
├── task_03_detail_update_api.md
└── README.md (optional)
```

### 5.2 Naming Convention

| Task Type | Format | Example |
|-----------|--------|---------|
| Task 00 | `task_00_shared_data_model.md` | Always this name |
| Business | `task_{XX}_{name}.md` | `task_01_create_api.md` |

**Rules**:
- `{XX}` = Two-digit: 01, 02, 03
- `{name}` = lowercase with underscores: `create_api`, `list_api`
- `{feature_name}` = snake_case from feature: `khai_bao_bhxh`

### 5.3 Feature Name Extraction

| Checklist Feature | Directory Name |
|-------------------|----------------|
| Khai báo BHXH | `khai_bao_bhxh` |
| Quản lý sự vụ | `quan_ly_su_vu` |
| Đối soát thu hộ | `doi_soat_thu_ho` |

---

## 6. Task 00 Template (Foundation)

```markdown
# Task 00: Shared Data Model - [Feature Name]

**File**: `docs/tasks/[feature_name]/task_00_shared_data_model.md`

## 1. Purpose
Define complete data model for [Feature Name].

## 2. Scope
- [ ] FULL entities (all database fields)
- [ ] Repository interfaces (CRUD + queries)
- [ ] Enums
- [ ] Error codes
- [ ] State machine (if applicable)

## 3. Entities

### EntityName
- [ ] Table: [table_name]
- [ ] Fields: [list ALL fields from database]
- [ ] Relationships: @OneToMany, @ManyToOne
- [ ] Annotations: @Table, @Column, @EntityListeners

## 4. Repositories

### EntityNameRepository
- [ ] Extends: JpaRepository<EntityName, Integer>
- [ ] Query methods: findByXxx(), existsByXxx()
- [ ] Custom queries: @Query (if needed)

## 5. Enums
- [ ] EnumName: VALUE_1, VALUE_2, VALUE_3

## 6. Error Codes
- [ ] ERROR_CODE_1 (HTTP status, "message")
- [ ] ERROR_CODE_2 (HTTP status, "message")

## 7. Dependencies
- [ ] Database schema exists

## 8. Estimate
- **Hours**: 8-12
- **Blocks**: All other tasks
```

---

## 7. Business Task Template

```markdown
# Task XX: [API Name] - [Feature Name]

**File**: `docs/tasks/[feature_name]/task_[XX]_[name].md`

## 1. Purpose
Implement [API description].

## 2. Scope
- [ ] [METHOD] /api/{app|portal}/v1/...
- [ ] RequestDTO: [Name]
- [ ] ResponseDTO: [Name]
- [ ] Service: [methodName]()
- [ ] Validation: [rules]
- [ ] Error handling

## 3. Checklist Coverage
- Checklist Section 2, item 2.X
- Checklist Section 4, items 4.X.X-4.X.Y
- Checklist Section 5, items 5.X-5.Y

## 4. Data Structures

### RequestDTO
| Field | Type | Validation |
|-------|------|------------|
| field1 | String | @NotNull, @Size(max=100) |
| field2 | Integer | @Min(0) |

### ResponseDTO
| Field | Type | Source |
|-------|------|--------|
| id | Integer | Entity |
| calculatedField | BigDecimal | Business logic |

## 5. Business Logic (Pseudocode)

```
FUNCTION [methodName](request):
1. Validate input (@Valid in controller)
2. Cross-field validation
3. Business rules validation
4. Query repository
5. Build/update entity
6. Save entity
7. Build response
8. Log event
9. RETURN response
```

## 6. Implementation Checklist
- [ ] Create RequestDTO with validation
- [ ] Create ResponseDTO
- [ ] Implement service method with pseudocode
- [ ] Create controller endpoint
- [ ] Add error handling
- [ ] Write tests

## 7. API Contract
| Method | Path | Request | Response | Errors |
|--------|------|---------|----------|--------|
| POST | /api/... | RequestDTO | ResponseDTO | ERROR_1, ERROR_2 |

## 8. Acceptance Criteria
- [ ] API accessible at correct path
- [ ] Valid request returns 200
- [ ] Validation returns 400
- [ ] Business logic correct

## 9. Dependencies
- [ ] Task 00 ✅

## 10. Estimate
- **Hours**: X
- **Complexity**: Low/Medium/High
```

---

## 8. Step-by-Step Process

| Step | Action | Output |
|------|--------|--------|
| **1** | Read checklist QC | Understand all 10 sections |
| **2** | Detect parallel features | Identify N independent features |
| **3** | Extract Task 00 | List ALL entities, repos, enums, errors |
| **4** | Apply splitting rules | Decide task boundaries |
| **5** | Apply grouping rules | Group related items |
| **6** | Order by business flow | Create → List → Detail → Update → Delete |
| **7** | Create directory | `docs/tasks/{feature_name}/` |
| **8** | Create task files | N files using templates |
| **9** | Validate completeness | 100% checklist coverage |
| **10** | Create README (optional) | Task index with dependencies |

---

## 9. Quality Criteria

| Criteria | Check |
|----------|-------|
| **Coverage** | All checklist items → task |
| **Traceability** | Each task references checklist |
| **File Structure** | Files in `docs/tasks/{feature_name}/` |
| **Naming** | `task_XX_{name}.md` format |
| **Independence** | Tasks can be developed in parallel |
| **Size** | Each task ≤16 hours |

---

## 10. Common Patterns

### Pattern 10.1: Simple CRUD (5 endpoints)

**Input**: Checklist with 1 entity, 5 APIs, 10 validations

**Output**:
```
docs/tasks/khai_bao_bhxh/
├── task_00_shared_data_model.md    (Entity: 20 fields, Repository)
├── task_01_create_list_api.md      (2 APIs)
├── task_02_detail_update_api.md    (2 APIs)
└── task_03_delete_api.md           (1 API)
```

### Pattern 10.2: Complex Feature (7 endpoints + integration)

**Input**: Checklist with CRUD + Export + VSS integration

**Output**:
```
docs/tasks/doi_soat/
├── task_00_shared_data_model.md
├── task_01_create_list_api.md
├── task_02_detail_update_api.md
├── task_03_helper_apis.md
├── task_04_export_excel.md         (POI skill)
└── task_05_vss_integration.md      (WebClient skill)
```

### Pattern 10.3: Multi-Feature (2 parallel tracks)

**Input**: Checklist with 2 user stories, different API prefixes

**Output**:
```
docs/tasks/doi_soat/
├── Track A (Fee):
│   ├── task_00_shared_data_model.md
│   ├── task_01_create_list.md
│   └── task_02_detail_update.md
└── Track B (Daily):
    ├── task_00_shared_data_model.md  (or shared)
    └── task_05_create_list.md
```

---

## 11. Anti-Patterns (Avoid)

| Anti-Pattern | Why Bad | Correct Approach |
|--------------|---------|------------------|
| **Horizontal slicing** | High coupling | Vertical slice per task |
| **Thin foundation** | Entity conflicts | Task 00 has FULL entities |
| **Wrong directory** | No organization | Use `docs/tasks/{feature}/` |
| **Generic names** | Unclear purpose | Use descriptive names |
| **One section = One task** | May be wrong size | Group by cohesion |

---

## 12. Quick Reference

### Task Size Guidelines

| Checklist Items | Estimated Tasks |
|-----------------|-----------------|
| 5 endpoints | 3-4 tasks |
| 10 endpoints | 5-7 tasks |
| 2 features | 2 tracks (parallel) |
| Complex export | +1 task (separate skill) |

### Naming Examples

| API/Feature | Task File Name |
|-------------|----------------|
| Create hồ sơ | `task_01_create_api.md` |
| List & Search | `task_02_list_api.md` |
| Detail & Update | `task_03_detail_update_api.md` |
| Export Excel | `task_04_export_excel.md` |
| VSS Integration | `task_05_vss_integration.md` |

### Dependency Examples

```
Task 01: No dependencies
Task 02: Depends on Task 00 ✅
Task 03: Depends on Task 00 ✅
Task 04: Depends on Task 01 ✅ (uses Create response)
```

---

## 13. Checklist Validation

**Before completing task breakdown, verify**:

- [ ] All 10 checklist sections mapped
- [ ] Task 00 created with FULL entities
- [ ] Business tasks have DTOs (not entities)
- [ ] Files in correct directory: `docs/tasks/{feature_name}/`
- [ ] File names follow: `task_XX_{name}.md`
- [ ] Each task ≤16 hours
- [ ] Dependencies documented
- [ ] Parallel tracks identified (if applicable)

---

**End of Rule**

Version: 3.0 (Concise)
Created: 2026-02-20
Based on: rule_create_checklist.md v4.0

**Key Principles**:
1. Task 00 = Complete data model (FULL entities, repos, enums, errors)
2. Business tasks = API implementation (DTOs, services, controllers)
3. File structure = Feature-based directories
4. Task size = 8-16 hours, vertical slice
5. Naming = task_XX_{descriptive_name}.md
