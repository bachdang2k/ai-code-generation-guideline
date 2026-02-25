# Hướng Dẫn Quy Trình Phát Triển Backend Java

> **Phiên bản**: 1.0  
> **Ngày tạo**: 2026-02-24  
> **Ngôn ngữ**: Vietnamese  

---

## Mục Lục

- [Phần 1: Tổng quan về Workflow](#phần-1-tổng-quan-về-workflow)
- [Phần 2: Cách tạo Workflow, Rules và Skills](#phần-2-cách-tạo-workflow-rules-và-skills)
  - [2.1 Ví dụ quy trình tạo Skill](#21-ví-dụ-quy-trình-tạo-skill)
  - [2.2 Ví dụ quy trình tạo Workflow](#22-ví-dụ-quy-trình-tạo-workflow)
  - [2.3 Ví dụ quy trình tạo Rule](#23-ví-dụ-quy-trình-tạo-rule)
- [Phần 3: Ví dụ thực tế - Feature Đối soát thù lao](#phần-3-ví-dụ-thực-tế---feature-đối-soát-thù-lao)

NOTE: BẠN CẦN LÊN TRƯỚC PHƯƠNG ÁN KỸ THUẬT, CẦN PHẢI CÓ THIẾT KÊ DATABASE, FLOW CODE CŨNG CẦN PHẢI LÊN Ý TƯỞNG TRƯỚC. SAU ĐÓ AI SẼ HỖ TRỢ CHÚNG TA LÀM CHI TIẾT Ý TƯỞNG ĐÓ TRƯỚC KHI BIẾN Ý TƯỞNG THÀNH CODE.
---

## Phần 1: Tổng quan về Workflow

### 1.1 Workflow là gì?

Workflow (quy trình) là một chuỗi các bước có trình tự, được thiết kế để đảm bảo chất lượng và tính toàn vẹn của phần mềm được phát triển. Với dự án BHXH Backend, workflow bao gồm 7 giai đoạn chính:

**📖 Chi tiết Workflow**: Xem [`rule-base/work_flow_detail.md`](rule-base/work_flow_detail.md)

**📖 Chi tiết Workflow (Tiếng Việt)**: Xem [`rule-base/work_flow_Vietnamese`](rule-base/work_flow_Vietnamese.md)

### 1.2 Tại sao cần Workflow?

- ✅ **Đảm bảo chất lượng**: Mỗi bước đều có tiêu chuẩn rõ ràng (quality gates)
- ✅ **Tránh bỏ sót yêu cầu**: Checklist chi tiết bao phủ mọi kịch bản
- ✅ **Giảm thiểu lỗi**: Review và kiểm tra tại mỗi giai đoạn
- ✅ **Tính nhất quán**: Mọi feature đều phát triển theo quy trình giống nhau
- ✅ **Dễ dàng bảo trì**: Tài liệu đầy đủ, dễ dàng tra cứu sau này
- ✅ **Phát hiện sớm**: Issues được phát hiện sớm, chi phí sửa thấp hơn

### 1.3 Các bước trong Workflow và Ý nghĩa

| Giai đoạn | Tên bước | Ý nghĩa | Tại sao cần? |
|-----------|----------|---------|--------------|
| **Phase 1** | **Brainstorming** | Hiểu yêu cầu, làm rõ scope, thiết kế giải pháp | Tránh hiểu sai, tránh làm over-scope, đảm bảo mọi bên cùng hiểu về yêu cầu |
| **Phase 2** | **Acceptance Criteria** | Xây dựng checklist QC với 11 section đầy đủ | Đảm bảo bao quát mọi kịch bản (happy path, edge cases, error, permissions...), có cơ sở để test và verify |
| **Phase 3** | **PAKT (Phương án kỹ thuật)** | Định nghĩa giải pháp kỹ thuật chi tiết | Đảm bảo kiến trúc đúng, data model đúng, API design đúng, tránh phải refactor lớn sau này |
| **Phase 4** | **Task Breaking** | Phân nhỏ PAKT thành các task executable | Giúp AI code generation hiệu quả hơn, mỗi task có scope rõ ràng, dễ tracking progress |
| **Phase 5** | **AI Generate Code** | Thực thi các task để sinh code | Tự động hóa coding, giảm công việc lặp lại, đảm bảo consistency với codebase |
| **Phase 6** | **Fix & Improve** | Chạy test, fix lỗi, review code | Đảm bảo code chạy đúng, tuân thủ code style, không có security issues |
| **Phase 7** | **Update Docs** | Cập nhật tài liệu cuối cùng | Có đầy đủ tài liệu để tra cứu, handover, bảo trì sau này |

### 1.4 Rules là gì và tại sao cần?

**Rules** là các quy tắc/hướng dẫn chi tiết để thực hiện từng bước trong workflow.

**Tại sao cần Rules?**
- 📋 **Định nghĩa chuẩn**: Cấu trúc tài liệu được chuẩn hóa (checklist 11 sections, PAKT 10 sections)
- 🎯 **Tránh bỏ sót**: Rules nhắc các điểm cần kiểm tra (VD: FK đến dm_* dùng `ma` không dùng `id`)
- 🔁 **Có thể tái sử dụng**: Mỗi lần làm feature mới chỉ việc apply rule
- 🤖 **Giúp AI làm tốt hơn**: Prompt rõ ràng → AI sinh ra kết quả chất lượng hơn
- 📚 **Kiến thức được tài liệu hóa**: Best practices được lưu lại trong rules

**Các Rules chính trong dự án**:
- [`rule_create_checklist.md`](rule-base/rule_create_checklist.md) - Tạo checklist QC
- [`rule_create_pakt.md`](rule-base/rule_create_pakt.md) - Tạo PAKT
- [`rule_break_tasks_from_pakt.md`](rule-base/rule_break_tasks_from_pakt.md) - Phân nhỏ task

### 1.5 Skills là gì và tại sao cần?

**Skills** là các kỹ năng/hướng dẫn cụ thể cho AI Agent để thực hiện các nhiệm vụ phức tạp.

**Tại sao cần Skills?**
- 🧠 **Đóng gói logic phức tạp**: Brainstorming skill định nghĩa cách tư duy từ requirements → design
- 🎯 **Step-by-step guidance**: Executing Plans skill hướng dẫn cách chạy task batches
- 🔄 **Tái sử dụng**: Skills có thể dùng cho nhiều feature khác nhau
- 📈 **Cải thiện liên tục**: Skills được update dần theo kinh nghiệm thực tế

**Các Skills chính**:
- [Brainstorming Skill](.agent/skills/brainstorming/SKILL.md) - Giai đoạn 1
- [Executing Plans Skill](.agent/skills/executing-plans/SKILL.md) - Giai đoạn 5

---

## Phần 2: Cách tạo Workflow, Rules và Skills

### 2.1 Ví dụ quy trình tạo Skill

Skills là các "kỹ năng" giúp AI Agent thực hiện nhiệm vụ phức tạp. Quy trình tạo Skill như sau:

#### Research opensource patterns

Bắt đầu từ việc nghiên cứu các opensource templates và patterns từ cộng đồng, ví dụ:
- **Superpowers** (https://github.com/obra/superpowers) - Framework để tạo AI skills
- Các coding assistant frameworks khác
- Best practices từ industry

#### Customize cho project context

Sau khi nghiên cứu, customize skills cho phù hợp với BHXH Backend:

**Context cần thêm**:
- Tech stack: Spring Boot 3.4.4, Java 21, MySQL 8, Flyway, Redis
- Code conventions: snake_case DB, camelCase Java, Vietnamese naming cho business logic
- Critical rules: FK mapping (dm_* dùng 'ma'), @PreAuthorize cho CMS, @Transactional cho writes
- Workflow 7 phases của dự án

**Ví dụ Skills trong dự án**:

| Skill | Mục đích | Location |
|-------|----------|----------|
| **Brainstorming Skill** | Giúp AI tư duy từ requirements → design | [`.agent/skills/brainstorming/SKILL.md`](.agent/skills/brainstorming/SKILL.md) |
| **Executing Plans Skill** | Giúp AI chạy task batches hiệu quả | [`.agent/skills/executing-plans/SKILL.md`](.agent/skills/executing-plans/Skill.md) |

**Quy trình ngắn gọn**:
```
1. Research opensource templates (Superpowers, etc.)
2. Chọn pattern phù hợp (VD: brainstorming, planning)
3. Custom với BHXH Backend context
4. Tạo file .agent/skills/{name}/SKILL.md
5. Test và refine (1-2 iterations)
```

---

### 2.2 Ví dụ quy trình tạo Workflow

Workflow của dự án được phát triển dựa trên kinh nghiệm thực tế và quy trình phát triển phần mềm đã được tinh chỉnh qua nhiều projects.

#### Nguồn cảm hứng từ Software Development Life Cycle (SDLC)

Tích hợp các yếu tố từ:
- **Agile/Scrum**: Iterative development, feedback loops
- **Waterfall**: Phase-by-phase approach với quality gates
- **DevOps**: CI/CD, continuous improvement
- **Document-driven Development**: Tài liệu là "single source of truth"

#### Quy trình custom 7 phases

```
┌─────────────────────────────────────────────────────────────┐
│  1. Discovery & Brainstorming                               │
│  • Nghiên cứu URD, Figma, Database schema                   │
│  • Hiểu business requirements                               │
│  • Làm rõ scope và assumptions                              │
│  Output: Design proposal được stakeholder approve           │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  2. Requirements Definition (Checklist QC)                  │
│  • Định nghĩa acceptance criteria với 11 sections           │
│  • Bao phủ: Happy paths, Common scenarios, Validations...   │
│  Output: Checklist document đầy đủ                          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Technical Design (PAKT)                                 │
│  • Architecture, API, Data model, Business logic            │
│  • Map checklist items sang giải pháp kỹ thuật              │
│  Output: PAKT document                                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Decomposition (Task Breaking)                           │
│  • Phân nhỏ PAKT thành executable tasks                     │
│  • Mỗi task có scope rõ ràng, có code references            │
│  Output: Task solution files                                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  5. Implementation (AI Generate Code)                       │
│  • Execute tasks theo batches                               │
│  • Verify sau mỗi batch                                     │
│  • Handle blockers                                          │
│  Output: Source code                                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Verification (Fix & Improve)                            │
│  • Run tests, lint, typecheck                               │
│  • Code review, security check                              │
│  Output: Clean, tested code                                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Documentation & Handover                                │
│  • Update PAKT với implementation status                    │
│  • Update Checklist với verification results                │
│  Output: Complete documentation package                     │
└─────────────────────────────────────────────────────────────┘
```

#### Decision Points & Feedback Loops

**Quality Gates** (giữa các phases):
- Mỗi phase phải pass quality gate mới được proceed
- Failed → Return về phase đó để fix

**Feedback Loops**:
- Brainstorming: Design not approved → Re-brainstorm
- Checklist: Sections incomplete → Complete missing parts
- PAKT: Review fail → Update PAKT
- Code generation: Compile fail → Fix và retry
- Verification: Tests fail → Fix và re-test

#### Iterative Improvement

Workflow không cố định mà được cải thiện qua:
- **Trải nghiệm thực tế**: Áp dụng vào nhiều projects
- **Feedback từ team**: Thu thập ý kiến và điều chỉnh
- **Lessons learned**: Document các vấn đề gặp phải và cách tránh
- **Best practices**: Thu thập patterns từ industry và adapt cho dự án

---

### 2.3 Ví dụ quy trình tạo Rule

Quy trình tạo Rule đã được tinh chỉnh qua nhiều iterations và áp dụng thành công cho dự án BHXH Backend.

#### 2.3.1 Triết lý tạo Rules (6-step Process)

```
┌─────────────────────────────────────────────────────────────┐
│  Bước 1: Xác định core points của tài liệu cần tạo          │
│  (VD: Checklist cần 11 section, PAKT cần 10 section)        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Bước 2: Generate prompt tối ưu từ core points              │
│  • Dựa vào cấu trúc tài liệu                                │
│  • Thêm best practices từ ngành                             │
│  • Kết hợp với kinh nghiệm coding                           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Bước 3: Kết hợp với Project Context để optimize tiếp       │
│  • Công nghệ stack (Spring Boot, Java 21...)                │
│  • Convention của dự án (snake_case DB, camelCase Java...)  │
│  • Universal rules (FK mapping, security...)                │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Bước 4: Sử dụng Chatbot để refine prompt                   │
│  • Chatbot giúp tối ưu wording                              │
│  • Thêm các góc nhìn bị bỏ sót                              │
│  • Make prompt clearer và more actionable                   │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Bước 5: Đưa prompt đã optimize cho AI Agent                │
│  • Agent sinh ra rule draft                                 │
│  • Kết quả thường ở khoảng 70-80% hoàn thiện                │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Bước 6: Review → Experience → Update (Lặp lại)             │
│  • Review rule vừa tạo                                      │
│  • Áp dụng vào thực tế, tìm điểm chưa tốt                   │
│  • Cập nhật rule theo kinh nghiệm                           │
│  • Lặp lại cho đến khi hài lòng (thường 3-5 iterations)     │
└─────────────────────────────────────────────────────────────┘
```

#### 2.3.2 Ví dụ: Tạo Rule Checklist QC

**Bước 1: Xác định core points**
- Checklist cần bao phủ đầy đủ scenarios
- Cần có structure để đảm bảo không bỏ sót
- Cần testable

**Bước 2: Generate prompt ban đầu**
```markdown
Create a rule for generating Quality Checklist for software features.
The checklist should have sections covering:
- Happy paths, Common scenarios, Validations
- Business rules, Error handling, Edge cases
- Permissions, External integrations, Definition of Done
```

**Bước 3: Kết hợp Project Context**
```markdown
Add project-specific context:
- Backend: Spring Boot 3.4.4, Java 21, MySQL 8
- Database: snake_case convention
- API: CMS (/api/v1/**) vs App (/api/app/v1/**)
- Security: @PreAuthorize for CMS
- FK mapping: dm_* tables use 'ma' not 'id'
```

**Bước 4: Chatbot refine prompt**
Chatbot tối ưu hóa với:
- Clear structure definition
- Specific examples for each section
- Coverage targets (100% happy paths, 50-70% common)
- Traceability requirements

**Bước 5: Agent tạo rule**
Kết quả: [`rule_create_checklist.md`](rule-base/rule_create_checklist.md)

**Bước 6: Review và update**
- Version 1: Chỉ 9 sections
- Applied thử → Thiếu "Non-functional"
- Update → Thêm section 10
- Applied thử → Cần "Quality Gates" rõ hơn
- Update → Thêm section 11 "Definition of Done" với tiers
- Final: 11 sections với detailed guidance

#### 2.3.3 Ví dụ: Prompt tạo Rule Checklist (Hiện có)

Dưới đây là prompt được regenerate từ [`rule_create_checklist.md`](rule_base/rule_create_checklist.md):

```markdown
You are a Senior QA Engineer and Business Analyst creating a comprehensive Quality Checklist for a BHXH Backend feature.

# Context
- Project: BHXH Backend - Spring Boot 3.4.4, Java 21, MySQL 8
- Architecture: Layered (Controller → Service → Repository)
- API Types: CMS (/api/v1/**) requires @PreAuthorize, App (/api/app/v1/**) requires authentication
- Database: snake_case tables, FK to dm_* uses 'ma' not 'id'

# Task
Create a checklist document named: checklist_qc_{app|portal}_{featurename}.md

# Structure - 11 Sections

## Section 1: Feature Overview
- Feature name and scope
- URD references
- Business context

## Section 2: Happy Path Scenarios (100% Required)
- Main success scenarios
- All valid input permutations
- Complete user journeys
- Use Given-When-Then format

## Section 3: Common Use Scenarios (50-70% Coverage)
- Typical user behaviors
- Most frequent use cases
- Expected partial failures

## Section 4: Validation Rules
- All input field validations
- Business rule validations
- Permission checks

## Section 5: Business Rules
- Calculations and formulas
- State transition rules
- Constraints and invariants

## Section 6: State Transitions
- Initial states
- Valid state transitions
- State queries

## Section 7: Error Scenarios
- Expected error cases
- Error codes and messages
- Recovery behaviors

## Section 8: Edge Cases
- Boundary conditions
- Empty/Null scenarios
- Concurrent operations

## Section 9: Permissions & Access
- CMS permission requirements
- App authentication requirements
- Data scoping (Admin vs User)

## Section 10: External Capabilities
- External API calls
- Timeout scenarios
- Export capabilities

## Section 11: Definition of Done
- MANDATORY TIER 1: Happy paths (100%)
- MANDATORY TIER 2: Common scenarios (100%)
- MANDATORY: Core requirements (100%)
- ENHANCED: Error scenarios (optional)
- BONUS: Edge cases (optional)

# Quality Gates
- All User Stories have Happy Path Scenarios
- All scenarios traceable to URD (Ref: US-XX)
- Testable criteria (not "user-friendly" without specifics)
```

#### 2.3.4 Ví dụ: Tạo Rule PAKT

**Core points**:
- Technical solution với architecture, API, data model, business logic, integration, security

**Project context**:
- Spring Boot patterns (Controller → Service → Repository)
- JPA entities with FK mapping rules
- Service layer patterns (@Transactional, Event publishing)

**Final**: [`rule_create_pakt.md`](rule-base/rule_create_pakt.md)

#### 2.3.5 Ví dụ: Tạo Rule Break Tasks

**Core points**:
- Task breakdown cho AI execution
- Cần templates và code references

**Project context**:
- Task structure: Data Model (Task 00) → API (Task 01+) → Business Logic (Task 02+) → Integration (Task 03+)
- Code examples từ codebase hiện có

**Final**: [`rule_break_tasks_from_pakt.md`](rule_base/rule_break_tasks_from_pakt.md)

#### 2.3.6 Key Takeaways

✅ **Start small**: Bắt đầu với draft đơn giản  
✅ **Context is king**: Luôn kết hợp với project context  
✅ **Use tools**: Chatbot giúp optimize prompt  
✅ **Iterate**: Review → Update → Review again  
✅ **Document lessons learned**: Mỗi iteration đều record lại lessons  

---

## Phần 3: Ví dụ thực tế - Feature Đối soát thù lao

### 3.1 Overview

Feature **Đối soát thù lao** là feature cho phép Đại lý thu tổng hợp và xác nhận thù lao thu hộ từ các đợt kê khai đã được CQ BHXH xử lý.

**Scope**: 5 User Stories (US-TL01 → US-TL05)
- US-TL01: Xem danh sách đối soát
- US-TL02: Tạo đối soát
- US-TL03: Xem chi tiết, Chốt, Xuất file
- US-TL04: Sửa đối soát
- US-TL05: Xóa đối soát

Sau khi đã overview URD, figma tài liệu: 

BẠN CẦN LÊN TRƯỚC PHƯƠNG ÁN KỸ THUẬT, CẦN PHẢI CÓ THIẾT KÊ DATABASE, FLOW CODE CŨNG CẦN PHẢI LÊN Ý TƯỞNG TRƯỚC. SAU ĐÓ AI SẼ HỖ TRỢ CHÚNG TA LÀM CHI TIẾT Ý TƯỞNG ĐÓ TRƯỚC KHI BIẾN Ý TƯỞNG THÀNH CODE.

### 3.2 Giai đoạn 1: Brainstorming

**Prompt**: I want using workflow define in @rule-base/work_flow_base.md; first brainstorming URD define in @URD/portal/Đối soát/Đối soát thù lao/ 

**File output**: [`brainstorming_output/brainstorming_doi_soat_thu_lao.md`](brainstorming_output/brainstorming_doi_soat_thu_lao.md)

**Input**: User Requirements trong thư mục [`URD/portal/Đối soát/Đối soát thù lao/`](URD/portal/Đối soát/Đối soát thù lao/)

**Kết quả**:
- Hiểu rõ business flow: Đại lý chọn đợt kê khai → Hệ thống tổng hợp → Tính thù lao → Chốt để xác nhận
- Làm rõ actors: Admin (xem tất cả), Đại lý thu (chỉ xem của mình)
- Xác định data cần: Đợt kê khai, Hồ sơ, Tỷ lệ thù lao, etc.


### 3.3 Giai đoạn 2: Acceptance Criteria Checklist

**Prompt**: confirm Option A. Bây giờ tiến hành Phase 2: Acceptance Criteria Checklist. Để làm đúng theo workflow, bạn cần đọc rule tạo checklist trước. @rule-base/rule_create_checklist.md 

**File output**: [`docs/tasks/checklist_qc_portal_doisoat_thulao.md`](docs/tasks/checklist_qc_portal_doisoat_thulao.md)

**Nội dung chính**:
- **11 sections** đầy đủ theo rule template
- **Happy paths** (100%): 5 user stories × 3-5 scenarios mỗi story
- **Common scenarios** (50-70%): Search, filter, pagination, etc.
- **Validations**: Tên đối soát unique, giai đoạn không chọn ngày tương lai, etc.
- **Business rules**: Tính thù lao = Số tiền × Tỷ lệ, 1 hồ sơ chỉ thuộc 1 kỳ
- **State transitions**: DANG_DOI_SOAT → DA_DOI_SOAT
- **Permissions**: Admin (all) vs Đại lý thu (own only)
- **Definition of Done**: Tiers (Mandatory, Enhanced, Bonus)

**Review points**:
- ✅ Bao phủ hết 5 URDs
- ✅ Traceable (mỗi scenario có Ref: US-TLxx)
- ✅ Testable (specific, không có criteria mơ hồ như "user-friendly")

### 3.4 Giai đoạn 3: PAKT (Phương án kỹ thuật)

**Prompt**: Nice now from this checklist, create solutions docs follow rule @rule-base/rule_create_pakt.md 

**File output**: [`pakt/doi_soat_thu_lao/pakt_portal_doisoat_thulao.md`](pakt/doi_soat_thu_lao/pakt_portal_doisoat_thulao.md)

**Nội dung chính**:

#### Section 1: Overview & Scope
- Feature summary, business context, technical context
- In scope: CRUD, Tính thù lao, Xuất C17-TS, Xuất XLSX
- Out scope: Cập nhật tỷ lệ, Thanh toán

#### Section 2: Acceptance Criteria Mapping
- Map mỗi scenario → endpoint
- Map validation → implementation
- Map business rule → logic

#### Section 3: Architecture
- Layer breakdown: Controller → Service → Repository
- Request flow cho từng use case
- State machine diagram

#### Section 4: API Design
- 10 endpoints đầy đủ
- Request/Response DTOs pseudocode
- FK resolution strategy (quan trọng: JOIN không N+1)

#### Section 5: Data Model
- Entity definitions: NvDoiSoatThuLao, NvDoiSoatThuLaoHoSo
- Snapshot strategy: lưu ti_le_thu_lao vào thời điểm tạo
- State transitions table

#### Section 6: Business Logic
- Pseudocode cho các methods chính
- N+1 prevention strategy
- Tính thù lao logic

#### Section 7: Integration
- Export C17-TS (PDF)
- Export XLSX
- Internal enrichment (tỷ lệ thù lao từ nv_ho_so_dang_ky)

#### Section 8: Security
- RBAC matrix
- Data scoping (Admin vs Đại lý thu)
- Permission annotations

#### Section 9: Impact
- Performance requirements
- DDL references
- Compatibility

#### Section 10: Implementation Scope
- Task breakdown checklist
- Definition of Done

**Review points**:
- ✅ Tất cả checklist items mapped
- ✅ FK mapping đúng: dm_* dùng 'ma' không dùng 'id'
- ✅ N+1 prevention rõ ràng
- ✅ State machine defined
- ✅ Security matrix đầy đủ

### 3.5 Giai đoạn 4: Task Breaking

**Prompt**: from this pakt you breaks to tasks follow  @rule-base/rule_break_tasks_from_pakt.md

**Thư mục output**: [`pakt/doi_soat_thu_lao/`](pakt/doi_soat_thu_lao/)

**Các task files được tạo**:

| Task file | Nội dung | Mục đích |
|-----------|----------|---------|
| **task_00_data_model_solution.md** | Entities, Repositories, Enums, Error codes | Định nghĩa data model và foundation |
| **task_01_list_create_solution.md** | GET list API, POST create API | APIs cho US-TL01 và US-TL02 |
| **task_02_detail_chot_sua_xoa_solution.md** | GET detail, POST chot, PUT sửa, DELETE xóa | APIs cho US-TL03, US-TL04, US-TL05 |
| **task_03_export_solution.md** | Export C17-TS PDF, Export XLSX | Export capabilities cho US-TL03 |

**Đặc điểm mỗi task file**:
- ✅ Clear deliverables
- ✅ Code references từ codebase
- ✅ Pseudocode logic
- ✅ FK mapping examples (với `referencedColumnName = "ma"`)
- ✅ Validation examples

### 3.6 Giai đoạn 5-7: Implementation, Fix, Update Docs

**Prompt**: implement/update task in @pakt/doi_soat_thu_lao/ using skill @.agent/skills/executing-plans/SKILL.md 

Các giai đoạn này được thực hiện theo workflow:
- Phase 5: AI execute từng task file, verify sau mỗi batch
- Phase 6: Chạy test, fix lỗi, review code
- Phase 7: Update PAKT và Checklist với status cuối cùng

### 3.7 Lessons Learned từ feature này

**Điểm làm tốt** ✅:
- Checklist chi tiết giúp phát hiện sớm edge cases (BHYT không có tỷ lệ = 0%)
- PAKT có N+1 prevention strategy giúp avoid performance issues
- Task breaking rõ ràng giúp AI execute hiệu quả

**Điểm cần cải thiện** 🔄:
- Initial PAKT thiếu detail về N+1 prevention → đã update trong iteration 2
- Cần thêm example cho FK mapping → đã thêm trong task files

**Kinh nghiệm cho feature sau** 📚:
- Luôn kiểm tra N+1 risks khi design queries
- FK mapping rules cần nhắc lại trong mọi task
- Export logic cần pseudocode chi tiết hơn

---

## 4. Tổng kết

### 4.1 Key Takeaways

1. **Workflow đảm bảo chất lượng**: 7 phases với quality gates ở mỗi bước
2. **Rules là tài sản**: Rules được refine qua nhiều iterations, chứa best practices
3. **Context là quan trọng**: Luôn kết hợp rules với project context
4. **Iterate để hoàn thiện**: Không có rule nào hoàn thiện từ đầu, cần review và update
5. **Document everything**: Lessons learned được ghi lại để improve cho lần sau

### 4.2 References

| Tài liệu | Mô tả | Link |
|----------|-------|------|
| **Workflow chi tiết** | 7 phases với diagram, rules, examples | [`rule-base/work_flow_detail.md`](rule-base/work_flow_detail.md) |
| **Code Style Guidelines** | Coding conventions cho dự án | [`AGENTS.md`](AGENTS.md) |
| **Project Context** | System architecture và patterns | [`context/context.md`](context/context.md) |
| **Rules** | Rules để tạo checklist, PAKT, tasks | [`rule-base/`](rule-base/) |
| **Skills** | Brainstorming, Executing Plans | [`.agent/skills/`](.agent/skills/) |

### 4.3 Getting Started

Để bắt đầu với một feature mới:

1. **Đọc Workflow**: [`rule-base/work_flow_detail.md`](rule-base/work_flow_detail.md)
2. **Đọc Rules**: [`rule-base/`](rule-base/)
3. **Đọc Skills**: [`.agent/skills/`](.agent/skills/)
4. **Follow Phase 1-7**: Step by step, không skip phases
5. **Review và Improve**: Document lessons learned sau mỗi feature

---

*Tài liệu này sẽ được cập nhật liên tục khi có kinh nghiệm mới.*

**Version History**:
- v1.0 (2026-02-24): Initial version với 3 phần chính
