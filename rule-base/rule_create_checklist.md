# Rule: Generate Acceptance Criteria Checklist from URD

> **Version**: 5.0 (User-Focused, BDD-Enhanced)
> **Purpose**: Generate testable acceptance criteria (WHAT to build, not HOW)
> **Priority**: Happy paths + Common use scenarios mandatory (100%), edge cases optional
> **Format**: BDD scenarios + Rules-based validations
> **Usage Flow**: URD → Checklist → Solution Docs → Task Breaking → Acceptance Testing

---

## 📋 Table of Contents

1. [Quick Start Guide](#quick-start-guide)
2. [11-Section Structure](#11-section-structure)
3. [Section 1: Feature Overview](#section-1-feature-overview)
4. [Section 2: Happy Path Scenarios](#section-2-happy-path-scenarios--mandatory-100)
5. [Section 3: Common Use Scenarios](#section-3-common-use-scenarios--mandatory-50-70)
6. [Section 4: Validation Rules](#section-4-validation-rules)
7. [Section 5: Business Rules](#section-5-business-rules)
8. [Section 6: State Transitions](#section-6-state-transitions)
9. [Section 7: Error Scenarios](#section-7-error-scenarios--optional)
10. [Section 8: Edge Cases](#section-8-edge-cases--optional)
11. [Section 9: Permissions & Access](#section-9-permissions--access)
12. [Section 10: External Capabilities](#section-10-external-capabilities)
13. [Section 11: Definition of Done](#section-11-definition-of-done-)
14. [Quality Gates Checklist](#quality-gates-checklist)
15. [Key Principles](#key-principles)

---

## Quick Start Guide

### What to Do Before Asking Me to Code

**Step 1**: Read and understand this rule (5-10 minutes)
- Understand the 11-section structure
- Know which sections are mandatory (Sections 1-6, 9-11)
- Recognize the two-tier mandatory priority (Section 2 = 100%, Section 3 = 50-70%)

**Step 2**: Review your user requirement documents (URDs)
- Identify the feature name, scope (app/portal), and user stories
- Extract business context and value proposition

**Step 3**: Generate the checklist using the templates
- Copy Section 1 template (Feature Overview)
- Copy Section 2 template (Happy Path Scenarios) - **MUST BE COMPLETE**
- Copy Section 3 template (Common Use Scenarios) - **MUST BE COMPLETE**
- Copy other sections as needed

**Step 4**: Validate the checklist before showing it to stakeholders
- Run through the Quality Gates checklist (Section 14)
- Ensure all mandatory sections are populated
- Verify 100% happy path coverage

**Step 5**: Get approval and proceed to PAKT generation

---

## 11-Section Structure

| # | Section | Format | Priority | Coverage Target |
|---|---------|--------|----------|-----------------|
| 1 | Feature Overview | Summary | Mandatory | - |
| 2 | **Happy Path Scenarios** | **BDD (Given-When-Then)** | **MANDATORY ⭐** | **100%** (All success scenarios) |
| 3 | **Common Use Scenarios** | **BDD (Given-When-Then)** | **MANDATORY ⭐** | **50-70%** (Code branches) |
| 4 | Validation Rules | Rules-based | Mandatory | All fields |
| 5 | Business Rules | Rules-based | Mandatory | All rules |
| 6 | State Transitions | Rules-based | Mandatory | All states |
| 7 | Error Scenarios | Mixed | Optional | Enhanced |
| 8 | Edge Cases | Mixed | Optional | Bonus |
| 9 | Permissions & Access | Rules-based | Mandatory | All actions |
| 10 | External Capabilities | Rules-based | Mandatory | All integrations |
| 11 | Definition of Done | Checklist | Mandatory | Verification |

---

## Section 1: Feature Overview

**Purpose**: Provide context for acceptance criteria

**Template**:
```markdown
## Feature: [Name] ([App/Portal])

### Scope
- **User Stories**: US-01, US-02, US-03
- **Actors**: [List primary actors and roles]
- **Workflow**: [2-3 sentence summary of business workflow]

### URD References
- **Primary URD**: [file path]
- **Related URDs**: [file paths]

### Business Context
- **Feature Type**: [New feature / Enhancement / Bug fix]
- **Business Value**: [Why this feature matters to users]
```

**Example**:
```markdown
## Feature: Kê khai BHXH Tự nguyện (App)

### Scope
- **User Stories**: US-01, US-02, US-03, US-04
- **Actors**: Nhân viên thu hộ (creates profiles), Hệ thống (calculates, validates)
- **Workflow**: Agent enters participant info → System calculates payment → Agent saves → Profile awaits payment

### URD References
- **Primary URD**: /APP thu hộ/Kê khai BHXH/[US-01] Step 1.md, [US-02] Step 2.md
- **Related URDs**: /context/context.md (state machine, validation rules)

### Business Context
- **Feature Type**: New feature - Core BHXH collection workflow
- **Business Value**: Enables agents to collect voluntary social insurance from participants
```

---

## Section 2: Happy Path Scenarios ⭐ MANDATORY (100%)

**Purpose**: Primary success scenarios when everything works correctly

**Coverage**: 100% of all happy path scenarios - **MUST BE COMPLETE**

**Format**: Given-When-Then (BDD)

**Template**:
```markdown
## Happy Path Scenarios (MANDATORY - 100% Required) ⭐

### User Story: US-XX - [Title]

#### Scenario X.X: [Scenario title]
- [ ] Given: [Precondition 1]
- [ ] And: [Precondition 2]
- [ ] When: [User action]
- [ ] And: [Additional action]
- [ ] Then: [Expected outcome 1]
- [ ] And: [Expected outcome 2]
```

**Example**:
```markdown
## Happy Path Scenarios (MANDATORY - 100% Required) ⭐

### User Story: US-01 - Create BHXH Profile

#### Scenario 1.1: Agent creates profile with valid BHXH code
- [ ] Given: Agent is authenticated with valid account
- [ ] And: Participant has existing valid BHXH code (10 digits)
- [ ] When: Agent submits complete participant information (họ tên, CCCD, ngày sinh, giới tính)
- [ ] And: Agent submits payment details (phương thức đóng, số tháng, mức đóng)
- [ ] Then: System creates profile with unique mã hồ_so (format: BHXH-YYYY-XXX)
- [ ] And: Profile status is set to "CHỜ THU TIÊN"
- [ ] And: System determines method = "dong_tiep" (existing participant)
- [ ] And: System auto-calculates payment amount: số tháng × mức đóng × 22% - NSNN - NSĐP
- [ ] And: System displays success message with confirmation details

#### Scenario 1.2: Agent uses BHXH lookup to auto-fill participant info
- [ ] Given: Agent is on Step 1 (Kê khai thông tin người tham gia)
- [ ] And: Agent enters valid Mã số BHXH (10 digits)
- [ ] When: Agent clicks "Tra cứu" button
- [ ] Then: System calls BHXH API to lookup participant
- [ ] And: System auto-fills participant fields: họ tên, CCCD, ngày sinh, giới tính, địa chỉ
- [ ] And: Agent can review and edit auto-filled information
- [ ] And: System enables "Tiếp tục" button to proceed to Step 2
```

---

## Section 3: Common Use Scenarios ⭐ MANDATORY (50-70%)

**Purpose**: Important alternative paths and common business logic branches

**Coverage**: 50-70% of code branches/decision points - **MUST BE COMPLETE**

**Includes**:
- Alternative successful paths (different ways to achieve goal)
- Common validation recoveries (user fixes error and succeeds)
- Important business decision branches (method = X vs method = Y)

**Excludes**:
- Rare edge cases (concurrent access, boundary extremes)
- Technical errors (system failures, timeouts)

**Format**: Given-When-Then (BDD)

**Template**:
```markdown
## Common Use Scenarios (MANDATORY - 50-70% Code Branch Coverage) ⭐

### User Story: US-XX - [Title]

#### Scenario X.X: [Common scenario title]
- [ ] Given: [Preconditions]
- [ ] When: [User action/variation]
- [ ] Then: [Expected outcome]
```

**Example**:
```markdown
## Common Use Scenarios (MANDATORY - 50-70% Code Branch Coverage) ⭐

### User Story: US-01 - Create BHXH Profile

#### Scenario 1.3: Agent creates profile for NEW participant (no BHXH code)
- [ ] Given: Agent is authenticated
- [ ] And: Participant has no existing BHXH code
- [ ] When: Agent clicks "Tra cứu" with invalid code → System returns "Mã số BHXH không tồn tại"
- [ ] And: Agent submits complete participant information manually
- [ ] And: Agent submits payment details
- [ ] Then: System creates profile with unique mã hồ_so
- [ ] And: System determines method = "dang_ky_moi" (covers method decision branch)
- [ ] And: Profile status is "CHỜ THU TIÊN"

#### Scenario 1.4: Agent creates profile with NSĐP support
- [ ] Given: Agent is creating profile
- [ ] And: Province has NSĐP support available
- [ ] When: Agent selects "Tỷ lệ NSĐP hỗ trợ" = 15%
- [ ] Then: System auto-calculates "Số tiền NSĐP hỗ trợ" (covers NSĐP calculation branch)
- [ ] And: Payment amount reduced by NSĐP support amount

### User Story: US-02 - Calculate Payment Amount

#### Scenario 2.3: Agent selects MULTI-YEAR payment method
- [ ] Given: Agent is on Step 2 (Kê khai thông tin đóng bảo hiểm)
- [ ] When: Agent selects "Phương thức đóng" = "Đóng nhiều năm"
- [ ] And: Agent enters "Số tháng" = 36 (divisible by 12, within max 60)
- [ ] Then: System validates 36 % 12 = 0 (covers multi-year validation branch)
- [ ] And: System calculates payment using present value of annuity formula with discount rate (covers multi-year calculation branch)
- [ ] And: System displays final amount (rounded HALF_UP to VND unit)

#### Scenario 2.4: Agent selects MONTHLY payment method (1 month)
- [ ] Given: Agent is on Step 2
- [ ] When: Agent selects "Phương thức đóng" = "1 tháng"
- [ ] Then: System auto-fills "Số tháng" = 1 (read-only)
- [ ] And: System calculates using monthly formula: 1 × mức đóng × 22% - NSNN - NSĐP
- [ ] And: System displays final amount

#### Scenario 2.5: Agent enters maximum allowed payment amount
- [ ] Given: Agent is on Step 2
- [ ] And: Mức đóng tối đa in danh mục = 10,000,000 VND
- [ ] When: Agent enters mức đóng = 10,000,000 VND
- [ ] Then: System accepts the amount (covers max validation branch)
- [ ] And: System calculates payment correctly
- [ ] And: System enables "Tiếp tục" button
```

---

## Section 4: Validation Rules

**Purpose**: Document all validation rules in business language

**Format**: Rules-based checklist

**Template**:
```markdown
## Validation Rules

### Field-Level Validations (US-XX)
- [ ] **[Field Name]**: [Rule description] (Ref: US-XX)

### Cross-Field Validations (US-XX)
- [ ] **[Dependency description]**: [Rule] (Ref: US-XX)
```

**Example**:
```markdown
## Validation Rules

### Field-Level Validations (US-01)
- [ ] **Mã số BHXH**: Required, exactly 10 digits, number format (Ref: US-01)
- [ ] **Họ và tên**: Required, uppercase letters, Vietnamese character support (Ref: US-01)
- [ ] **Số CCCD**: Required, exactly 12 digits, number format (Ref: US-01)
- [ ] **Ngày sinh**: Required, format DD/MM/YYYY, must not be future date, age >= 15 years (Ref: US-01)
- [ ] **Giới tính**: Required, must be "Nam" or "Nữ" (Ref: US-01)
- [ ] **Số điện thoại**: Required, valid Vietnamese mobile network prefix (Ref: US-01)

### Field-Level Validations (US-02)
- [ ] **Cơ quan BHXH**: Required, dropdown, default assigned to agent (Ref: US-02)
- [ ] **Phương thức đóng**: Required, dropdown (1/3/6/12 months or multi-year) (Ref: US-02)
- [ ] **Số tháng**: Required
  - If monthly method: Auto-filled, read-only (1/3/6/12)
  - If multi-year: User input, positive integer, divisible by 12, max 60 (Ref: [Validation Rule: Multi-year Months])
- [ ] **Mức đóng**: Required, number, must be between min and max from danh mục (Ref: [Validation Rule: Payment Amount])

### Cross-Field Validations (US-02)
- [ ] **Method + Số tháng dependency**: If method = multi-year, số tháng must be divisible by 12, max 60 (Ref: [Business Rule: Payment Method])
- [ ] **Từ tháng + Đến tháng consistency**: Đến tháng = Từ tháng + Số tháng (auto-calculated, read-only) (Ref: US-02)
```

---

## Section 5: Business Rules

**Purpose**: Document business logic, calculations, and decision rules

**Format**: Rules-based checklist

**Template**:
```markdown
## Business Rules

### [Rule Category] (US-XX)
- [ ] **[Rule Name]**: If [condition], then [expected behavior] (Ref: US-XX)
```

**Example**:
```markdown
## Business Rules

### Method Determination (US-01)
- [ ] **Existing participant**: If participant has valid Mã số BHXH, then method = "dong_tiep" (Ref: [Business Rule: Method Determination])
- [ ] **New participant**: If participant has no Mã số BHXH, then method = "dang_ky_moi" (Ref: [Business Rule: Method Determination])

### Payment Calculations (US-02)
- [ ] **Monthly payment formula**: amount = số tháng × mức đóng × 22% - NSNN support - NSĐP support (Ref: [Business Rule: Payment Calculation])
- [ ] **Multi-year calculation**: Use present value of annuity formula with discount rate r (Ref: [Business Rule: Multi-year Calculation])
- [ ] **Rounding**: All amounts rounded to VND unit using HALF_UP (round 0.5 up) (Ref: [Business Rule: Payment Calculation])
- [ ] **Negative amount handling**: If calculated amount <= 0, display message: "NSNN hỗ trợ đủ chi trả" (Ref: [Business Rule: Payment Calculation])

### Profile Status Rules (All User Stories)
- [ ] **Initial status**: All newly created profiles have status = "CHỜ THU TIÊN" (Ref: [State Machine: Initial State])
- [ ] **Status transitions**: Only valid transitions allowed per state machine (Ref: [State Machine])
- [ ] **Incident blocking**: Profiles with active incidents (sự vụ) cannot change status (Ref: [Hard Rule: Incident Blocking])
```

---

## Section 6: State Transitions

**Purpose**: Document state machine rules for features with status/workflow states

**Format**: Rules-based checklist

**Template**:
```markdown
## State Transitions

### [Entity/Concept] State Machine (All User Stories)
- [ ] [Current State] → [Next State] (Event: [Trigger Event]) (Ref: [State Machine])

### Hard Rules
- [ ] [Hard rule description] (Ref: [Hard Rule: X])
```

**Example**:
```markdown
## State Transitions

### Hồ sơ State Machine (All User Stories)
- [ ] CHỜ THU TIÊN → ĐÃ THU_TIEN (Event: Xác nhận thu tiền by Admin) (Ref: [State Machine: Payment Confirmation])
- [ ] CHỜ THU TIÊN → ĐÃ HỤ (Event: Hủy hồ sơ by Admin) (Ref: [State Machine: Cancellation])
- [ ] ĐÃ THU_TIEN → CHỜ DUYỆT NỘI BỘ (Event: Nộp hồ sơ by Agent) (Ref: [State Machine: Submission])
- [ ] ĐÃ THU_TIEN → BỊ TỪ CHỐI (Event: Báo lỗi hồ sơ by Admin) (Ref: [State Machine: Rejection])
- [ ] CHỜ DUYỆT NỘI BỘ → CHỜ NỘP CQ BHXH (Event: Phê duyệt by Admin) (Ref: [State Machine: Approval])
- [ ] CHỜ DUYỆT NỘI BỘ → BỊ TỪ CHỐI (Event: Báo lỗi hồ sơ by Admin) (Ref: [State Machine: Rejection])
- [ ] CHỜ NỘP CQ BHXH → CQ BHXH ĐANG XỬ LÝ (Event: Ký số và nộp by Admin) (Ref: [State Machine: BHXH Submission])

### Hard Rules
- [ ] **Incident blocking**: Profiles with active incidents (sự vụ) in "Chờ duyệt" or "Đã duyệt" status cannot transition states (Ref: [Hard Rule: Incident Blocking])
- [ ] **Permission validation**: State transitions must validate user has required permission (Ref: [Hard Rule: Permissions])
- [ ] **Audit logging**: Every state change must publish audit event with user context (Ref: [Hard Rule: Audit Trail])
```

---

## Section 7: Error Scenarios ⚙️ OPTIONAL

**Purpose**: Document error scenarios and user-facing error messages

**Format**: Can use BDD or rules-based

**Completion**: ⚙️ OPTIONAL - Enhanced coverage

**Template**:
```markdown
## Error Scenarios (OPTIONAL - Enhanced Coverage) ⚙️

### [Error Type] (US-XX)
#### Scenario: [Scenario title]
- [ ] Given: [Preconditions]
- [ ] When: [User action causing error]
- [ ] Then: [System displays error message]
```

**Example**:
```markdown
## Error Scenarios (OPTIONAL - Enhanced Coverage) ⚙️

### Validation Errors (US-01)

#### Scenario: Agent submits invalid CCCD
- [ ] Given: Agent is creating new profile
- [ ] When: Agent enters CCCD with 11 digits (not 12)
- [ ] Then: System displays error: "Số CCCD không hợp lệ"
- [ ] And: System highlights CCCD field in red
- [ ] And: Profile is not created

#### Scenario: Agent submits future birth date
- [ ] Given: Agent is creating profile for child
- [ ] When: Agent enters ngày sinh = 01/01/2030 (future date)
- [ ] Then: System displays error: "Ngày sinh không hợp lệ (không được là ngày trong tương lai)"

### Business Errors (US-01)
- [ ] **Mã số BHXH not found**: If lookup returns no result, display "Mã số BHXH không tồn tại" (Ref: US-01)

### Business Errors (US-02)
- [ ] **Payment below minimum**: If mức đóng < minimum, display "Mức đóng thấp hơn mức tối thiểu cho phép" (Ref: [Validation Rule: Payment Amount])
- [ ] **Payment above maximum**: If mức đóng > maximum, display "Mức đóng vượt quá mức tối đa cho phép" (Ref: [Validation Rule: Payment Amount])
```

---

## Section 8: Edge Cases 🎯 OPTIONAL

**Purpose**: Document boundary conditions, rare scenarios, concurrent access

**Format**: Can use BDD or rules-based

**Completion**: 🎯 OPTIONAL - Bonus coverage

**Template**:
```markdown
## Edge Cases (OPTIONAL - Bonus Coverage) 🎯

### [Category] (US-XX)
- [ ] **[Edge case]**: [Expected system behavior] (Ref: US-XX)
```

**Example**:
```markdown
## Edge Cases (OPTIONAL - Bonus Coverage) 🎯

### Boundary Conditions (US-02)
- [ ] **Maximum payment**: Accept when mức đóng = 10,000,000 VND (maximum limit) (Ref: US-02)
- [ ] **Minimum payment**: Accept when mức đóng = 1,500,000 VND (minimum limit) (Ref: US-02)
- [ ] **Maximum multi-year months**: Accept when số tháng = 60 (divisible by 12) (Ref: [Validation Rule: Multi-year Months])

### Concurrent Access (US-HS03 - Update Profile)
- [ ] **Two agents update same profile**: Detect version conflict, display "Hồ sơ đã được cập nhật bởi người khác. Vui lòng làm mới và thử lại." (Ref: [Edge Case: Concurrency])
```

---

## Section 9: Permissions & Access

**Purpose**: Document who can perform what actions (business rules)

**Format**: Rules-based checklist

**Template**:
```markdown
## Permissions & Access

### [Interface/API Type] ([User Role])
- [ ] **[Action]**: [Permission requirement] (Ref: US-XX)

### Data Scoping
- [ ] **[Scope rule]**: [Description] (Ref: [Security Rule])
```

**Example**:
```markdown
## Permissions & Access

### App APIs (Nhân viên thu hộ - Collection Agent)
- [ ] **Create profile**: Authenticated user only (no special permission required) (Ref: US-01)
- [ ] **View profile list**: Agent can only see profiles they created (Ref: US-HS01, [Security Rule: Data Ownership])
- [ ] **View profile details**: Agent can only view profiles they created (Ref: US-HS02, [Security Rule: Data Ownership])
- [ ] **Update profile**: Agent can only update profiles they created (Ref: US-HS03, [Security Rule: Data Ownership])

### Portal APIs (Đại lý thu hộ / Admin)
- [ ] **View all profiles**: Admin can view all profiles within their agency scope (Ref: US-AHS01, [Security Rule: Agency Scope])
- [ ] **Approve profile**: Requires permission "BHXH_DUYET_HO_SO" (Ref: US-AHS05)
- [ ] **Reject profile**: Requires permission "BHXH_BAO_LOI_HO_SO" (Ref: US-AHS06)
- [ ] **Submit to BHXH Authority**: Requires permission "BHXH_NOP_CQ_BHXH" (Ref: US-CQBHXH02)

### Data Scoping
- [ ] **Agent data ownership**: Agents can only access profiles they created (Ref: [Security Rule: Data Ownership])
- [ ] **Admin agency scope**: Admins can access all profiles within their agency (Ref: [Security Rule: Agency Scope])
```

---

## Section 10: External Capabilities

**Purpose**: Document external system integrations (business needs)

**Format**: Rules-based checklist

**Template**:
```markdown
## External Capabilities

### [External System Name] (US-XX)
- [ ] **Capability**: [Business capability description]
- [ ] **Trigger**: [When system calls external system]
- [ ] **Error handling**: [What happens on failure] (Ref: US-XX)
```

**Example**:
```markdown
## External Capabilities

### BHXH Participant Lookup (US-01)
- [ ] **Capability**: Validate and retrieve participant information from BHXH system
- [ ] **Trigger**: Agent clicks "Tra cứu" button with Mã số BHXH
- [ ] **Expected**: System retrieves participant details and auto-fills form fields
- [ ] **Error - Not found**: Display "Mã số BHXH không tồn tại" (Ref: US-01)
- [ ] **Error - System unavailable**: Display "Không thể kết nối tới hệ thống BHXH. Vui lòng thử lại sau." (Ref: [External System: BHXH API])

### BHXH Authority Submission (US-CQBHXH02)
- [ ] **Capability**: Submit electronic declaration forms (D05-TS, D03-TS, TK01-TS) for processing
- [ ] **Trigger**: Admin completes "Ký số và nộp" workflow
- [ ] **Expected**: Forms transmitted, transaction code returned, profile statuses updated to "CQ BHXH ĐANG XỬ LÝ"
- [ ] **Error handling**: Display error message, profiles remain in "CHỜ NỘP CQ BHXH" (can retry) (Ref: US-CQBHXH02)
```

---

## Section 11: Definition of Done ⭐

**Purpose**: Verify completion per user story

**Two-Tier Mandatory Structure**:

**Template**:
```markdown
## Definition of Done (Per User Story) ⭐

### User Story: US-XX - [Title]

#### ✅ MANDATORY TIER 1: Happy Path Scenarios (100% Required)
- [ ] All happy path scenarios pass (Section 2, scenarios X.X)

#### ✅ MANDATORY TIER 2: Common Use Scenarios (100% Required)
- [ ] All common use scenarios pass (Section 3, covers 50-70% of code branches)

#### ✅ MANDATORY: Core Requirements (100% Required)
- [ ] Core validations enforced (Section 4)
- [ ] Business rules verified (Section 5)
- [ ] State transitions correct (Section 6, if applicable)
- [ ] Permissions validated (Section 9)
- [ ] External capabilities working (Section 10, if applicable)

#### ⚙️ ENHANCED: Error Scenarios (Optional)
- [ ] Error scenarios documented (Section 7)
- [ ] Error messages user-friendly (Vietnamese)

#### 🎯 BONUS: Edge Cases (Optional)
- [ ] Edge cases identified and handled (Section 8)

---
**DoD Status**: [ ] Complete (✅ Tier 1 + Tier 2 + Core = 100%, ⚙️ XX%, 🎯 XX%)
```

**Example**:
```markdown
## Definition of Done (Per User Story) ⭐

### User Story: US-01 - Create BHXH Profile

#### ✅ MANDATORY TIER 1: Happy Path Scenarios (100% Required)
- [ ] All happy path scenarios pass (Scenarios 1.1, 1.2 from Section 2)

#### ✅ MANDATORY TIER 2: Common Use Scenarios (100% Required)
- [ ] All common use scenarios pass (Scenarios 1.3, 1.4 from Section 3, covering method decision and NSĐP branches)

#### ✅ MANDATORY: Core Requirements (100% Required)
- [ ] Core validations enforced (CCCD 12 digits, ngày sinh <= today, age >= 15)
- [ ] Business rules verified (method determination: existing vs new participant)
- [ ] State correctly set to "CHỜ THU TIÊN" on creation
- [ ] Unique mã hồ_so generated with format BHXH-YYYY-XXX
- [ ] Permissions validated (authenticated users only, agent ownership)
- [ ] External capability validated (BHXH lookup)

#### ⚙️ ENHANCED: Error Scenarios (Optional)
- [ ] Validation error scenarios documented (invalid CCCD, future date)
- [ ] Error messages user-friendly in Vietnamese

#### 🎯 BONUS: Edge Cases (Optional)
- [ ] Edge cases identified (special Vietnamese characters)

---
**DoD Status**: [ ] Complete (✅ Tier 1 + Tier 2 + Core = 100%, ⚙️ 60%, 🎯 20%)

---

### User Story: US-02 - Calculate Payment Amount

#### ✅ MANDATORY TIER 1: Happy Path Scenarios (100% Required)
- [ ] All happy path scenarios pass (Scenarios 2.1, 2.2 from Section 2)

#### ✅ MANDATORY TIER 2: Common Use Scenarios (100% Required)
- [ ] All common use scenarios pass (Scenarios 2.3, 2.4, 2.5 from Section 3, covering multi-year calculation, monthly calculation, max validation branches)

#### ✅ MANDATORY: Core Requirements (100% Required)
- [ ] Core validations enforced (mức đóng within min/max, aligned with bước tăng, số tháng valid)
- [ ] Business rules verified (monthly formula, multi-year present value, rounding)
- [ ] Auto-calculations working (Đến tháng, NSNN support, NSĐP support, final amount)
- [ ] Payment amount displayed in red (highlighted)

#### ⚙️ ENHANCED: Error Scenarios (Optional)
- [ ] Validation error scenarios documented (below minimum, above maximum)
- [ ] Error messages specific and actionable

#### 🎯 BONUS: Edge Cases (Optional)
- [ ] Boundary cases tested (exact minimum, exact maximum)

---
**DoD Status**: [ ] Complete (✅ Tier 1 + Tier 2 + Core = 100%, ⚙️ 60%, 🎯 30%)
```

---

## Quality Gates Checklist {"#quality-gates-checklist"}

### Completeness Verification ⭐
- [ ] **User story coverage**: All user stories from URD have corresponding checklist items in Section 2 (Happy Paths)
- [ ] **Happy path completeness**: Happy path scenarios are 100% complete for ALL user stories (MANDATORY requirement)
- [ ] **Common use coverage**: Common use scenarios cover 50-70% of code branches for ALL user stories (MANDATORY requirement)
- [ ] **Mandatory sections populated**: Sections 1-6, 9-11 are populated (no empty mandatory sections)
- [ ] **Core validations documented**: All input field validations from URD are in Section 4
- [ ] **Business rules extracted**: All calculations, decisions, constraints from URD are in Section 5
- [ ] **State transitions defined**: If feature has state machine, all valid transitions are in Section 6
- [ ] **Permissions specified**: All user actions have permission requirements in Section 9
- [ ] **External capabilities identified**: If feature integrates with external systems, capabilities are in Section 10

### Quality Verification ⭐
- [ ] **No implementation details**: No API endpoints, HTTP methods, DTOs, database tables, framework annotations
- [ ] **Business/user language**: All items expressed in user-facing language (technical terms minimized)
- [ ] **Reference tags present**: Every item has reference tag (US-XX, [Business Rule], [State Machine], [Hard Rule])
- [ ] **Happy path priority**: Section 2 (Happy Paths) clearly marked as MANDATORY Tier 1
- [ ] **Common use priority**: Section 3 (Common Use Scenarios) clearly marked as MANDATORY Tier 2
- [ ] **Optional sections marked**: Sections 7-8 (Errors/Edges) marked as OPTIONAL/ENHANCED/BONUS
- [ ] **Scenarios testable**: All scenarios have clear pass/fail criteria (Given-When-Then format)
- [ ] **Vietnamese domain correct**: Business terms used correctly (hồ sơ, sự vụ, biên lai, trạng thái, etc.)

### Traceability Verification ⭐
- [ ] **URD to checklist mapping**: All URD user stories map to checklist items (100% traceability)
- [ ] **Workflow coverage**: All workflows from URD are covered in scenarios (Sections 2-3)
- [ ] **Validation traceability**: All validations from URD are documented (Section 4)
- [ ] **Error condition traceability**: All error conditions from URD are listed (Section 7 if populated)
- [ ] **No orphaned requirements**: All URD content is traced to checklist items (no missing requirements)

### DoD Verification ⭐
- [ ] **DoD per user story**: Section 11 has DoD checklist for EACH user story
- [ ] **Two-tier mandatory**: DoD clearly separates Tier 1 (Happy Paths) + Tier 2 (Common Use Scenarios) + Core Requirements
- [ ] **100% requirement explicit**: DoD explicitly states "100% Required" for both Tier 1 and Tier 2
- [ ] **Completion status trackable**: DoD includes status tracker (✅ Tier 1 + Tier 2 + Core = 100%, ⚙️ XX%, 🎯 XX%)

---

## Key Principles {"#key-principles"}

⭐ **Two-Tier Mandatory Priority**:
- **Tier 1 (MANDATORY)**: Happy Path Scenarios (Section 2) - 100% required
- **Tier 2 (MANDATORY)**: Common Use Scenarios (Section 3) - 100% required, covers 50-70% of code branches
- **Core Requirements (MANDATORY)**: Validations, rules, permissions, states - 100% required

⚙️ **Enhanced Coverage**: Sections 7-8 (Errors/Edges) are optional for enhanced quality

🚫 **No Implementation**: Exclude API endpoints, DTOs, database tables, framework annotations, technical details

✅ **Business Focus**: Use user-facing language, not technical implementation

📋 **Code Branch Coverage**: Section 3 (Common Use Scenarios) should cover 50-70% of business logic decision points

---

**End of Rule v5.0**
