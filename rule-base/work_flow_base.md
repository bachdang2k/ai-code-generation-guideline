# BHXH Backend Workflow

## Phase 1: Brainstorming
- Read: [Project Context](../context/context.md) and User requirement documents
- Use: [Brainstorming Skill](../../.agent/skills/brainstorming/SKILL.md)
- Steps: Initialize context → Understand idea → Clarify requirements → Validate understanding → Present design

## Phase 2: Acceptance Criteria Checklist
- Read: [Create Checklist Rule](rule_create_checklist.md)
- Create: docs/tasks/checklist_qc_{app|portal}_{featurename}.md
- Structure: 11 sections (Happy paths 100%, Common scenarios 50-70%, validations, rules, state, errors, edges, permissions, external, DoD)
- Verify: Completeness, Quality, Traceability, DoD checks

## Phase 3: PAKT (Technical Solution)
- Read: [Create PAKT Rule](rule_create_pakt.md)
- Create: pakt/pakt_{app|portal}_{featurename}.md
- Structure: 10 sections (Scope, mapping, architecture, API, data model, business logic, integration, security, impact, implementation)
- Verify: Pseudocode only, no implementation details, all scenarios mapped

## Phase 4: Task Breaking
- Read: [Break Tasks from PAKT Rule](rule_break_tasks_from_pakt.md)
- Create: pakt/{feature}/task_{XX}_{name}_solution.md
- Tasks: 00 (Data Model 8-12h), Business (8-16h), Integration (8-12h)
- Templates: Data model, API, Integration with code references

## Phase 5: AI Generate Code
- Read: [Executing Plans Skill](../../.agent/skills/executing-plans/SKILL.md)
- Execute: Batches of 3 tasks
- Steps: Load plan → Execute batch → Report → Continue → Complete
- Stop when blocked or verify fails

## Phase 6: Fixbug & Improvement
- Run: ./mvnw clean compile
- Run: ./mvnw test
- Check: [Code Style Guidelines](../../AGENTS.md) conventions, code quality, error handling, logging

## Phase 7: Update Final Documents
- Update: PAKT with implementation status
- Update: Checklist with verification
- Verify: All quality gates pass

## Universal Rules
- No DB schema changes without approval
- Error codes: 100000-400000
- BigDecimal for money, LocalDate/LocalDateTime for dates
- Vietnamese naming for business logic
- @Transactional for write operations
- @PreAuthorize for portal auth
- getCurrentUserRequired() for app auth
- BusinessException for errors
- Logging: log.info("Action: id={}, status={}", id, status)
- ./mvnw clean compile → ./mvnw test → ./mvnw

## References
- Skills: [Brainstorming Skill](../../.agent/skills/brainstorming/SKILL.md), [Executing Plans Skill](../../.agent/skills/executing-plans/SKILL.md)
- Rules: [Create Checklist Rule](rule_create_checklist.md), [Create PAKT Rule](rule_create_pakt.md), [Break Tasks from PAKT Rule](rule_break_tasks_from_pakt.md)
- Format: [Workflow Reference](../prompt/work_flow.md)
- Guidelines: [Code Style Guidelines](../../AGENTS.md)
