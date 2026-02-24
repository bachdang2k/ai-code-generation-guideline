# BHXH Backend - Business Domain & Workflow Context

> **Version**: 1.1
> **Purpose**: Concise business domain and workflow summary for reusable context in development sessions
> **Last Updated**: 2026-01-30

---

## Business Overview
(hide)

---

## Actors & Roles

(hide)
---

## Core Domain Concepts

(hide)

---

(hide)

### Important Rules (Hard Rules)

(hide)
---

## Business Workflows

(hide)

---

## Key Constraints & Invariants

### Hard Rules (Must Never Be Violated)
1. **Database Schema**: No CREATE/ALTER TABLE, DROP TABLE operations allowed
2. **State Machine**: Only valid transitions per state
3. **Incident Blocking**: Profiles with active incidents cannot proceed
4. **Permissions**: All admin operations must have @PreAuthorize checks
5. **Transaction Boundaries**: Business validation BEFORE @Transactional
6. **Error Codes**: Must use MessageResponseDict (Success = 0, Error = 100000-400000)
7. **Vietnamese Naming**: All business logic methods, messages, logs must be in Vietnamese
8. **Currency**: Always use BigDecimal for money
9. **Dates**: Use LocalDate/LocalDateTime, validate <= today for birth dates
10. **Audit Trail**: Every state change must log with user context
11. **Financial Consistency**: Receipt reissuance on incident approval with money changes
12. **History Tracking**: All significant events logged to Lịch sử hồ sơ

### Soft Rules (Configurable / Optional)
1. **Mức đóng**: Min/max values and Bước tăng configurable in danh mục
2. **Lãi suất**: For multi-year calculation, configurable per year
3. **Hospital Capacity**: Hospitals can be marked as at-capacity
4. **NSDP Support**: Province-level support rates, optional
5. **Batch Size**: 20 records/page for lists, but pagination is flexible

### Performance Requirements
1. **Pagination**: Default 20 records/page
2. **Batch Operations**: Use saveAll() for bulk updates
3. **Projections**: Use projection interfaces for list views to avoid N+1 queries
4. **Caching**: Use Redis for danh mục lookups (60-minute TTL default)
5. **Async Processing**: Digital signature and submission runs in background

---

## External Integrations

(hide)

---

## Database Design Principles

### Table/Column Naming
- **Prefixes**:
  - `dm_`: Danh mục (master data)
  - `ht_`: Hệ thống (system tables)
  - `nv_`: Nghiệp vụ (business data)
  - `ls_`: Lịch sử (history)
- **Naming**: Vietnamese without accents (snake_case) except ID, timestamp
- **Example**: `nv_ho_so_dang_ky`, `nv_su_vu`, `nv_bien_lai`, `nv_lich_su_ho_so`, `nv_dot_ke_khai`

### Foreign Key Relationships
- **Master data keys**: Use `ma` (VARCHAR) to sync with BHXH data
- **Business relationships**: Integer IDs

### JSON Columns
- **Flexible fields**: Use `mo_rong` (JSON) for dynamic data

### Constraints
- **Primary keys**: Auto-increment integers
- **Unique constraints**: On business keys (e.g., `ma_ho_so`, `ma_giao_dich`)
- **Indexes**: On frequently queried columns

### Key Business Tables
- `nv_ho_so_dang_ky`: Profiles (hồ sơ)
- `nv_su_vu`: Incidents (sự vụ)
- `nv_bien_lai`: Receipts (biên lai)
- `nv_lich_su_ho_so`: Record history (lịch sử hồ sơ)
- `nv_dot_ke_khai`: Declaration batches (đợt kê khai)
- `nv_to_khai_dien_tu`: Electronic forms (tờ khai điện tử)
- `nv_chi_tiet_bhxh`: BHXH details
- `nv_chi_tiet_bhyt`: BHYT details

---

## Technical Implementation Notes

(hide)

---

## Quick Reference - URD Location

```
(hide)
```

---

## Summary

This document provides:
- Business overview of system
- Core domain concepts 
- Complete state machine for profiles with all transitions
- Detailed workflows for all major features (8 workflows)
- Financial transaction management and reconciliation
- Validation rules per workflow step
- Hard vs. soft rules distinction
- Database design principles
- External integration points
- Technical implementation patterns

**Usage**: Reference this before any development work to understand business context, rules, and constraints.
