# Đối Soát Thù Lao — Task Solutions

**PAKT**: `pakt/doi_soat_thu_lao/pakt_portal_doisoat_thulao.md`  
**API Prefix**: `/api/portal/v1/doi-soat-thu-lao`  
**Created**: 2026-02-23  
**Status**: In Progress

## Overview

Feature quản lý kỳ đối soát thù lao thu hộ tại Portal. Bao gồm CRUD kỳ đối soát, tính thù lao từ `nv_ho_so_dang_ky.ti_le_thu_lao`, chốt kỳ (immutable), xuất C17-TS PDF và DS chi tiết XLSX.

**DB Schema**: V5 (all tables exist — no new migration needed)  
**Key Rule**: `ti_le_thu_lao` đọc từ `nv_ho_so_dang_ky.ti_le_thu_lao` (NULL → 0), không tra `nv_chinh_sach_thu_lao`  
**RBAC**: Admin (all agencies) | Đại lý thu (own `created_by_user_id` only)

---

## Task Dependency Matrix

| Task | ID | Name | Hours | Dependencies | Output | Status |
|------|----|------|-------|--------------|--------|--------|
| Task 00 | 00 | Data Model Foundation | 8h | None | Entities, Enums, Repos, State Machine, Error Codes | ⏳ |
| Task 01 | 01 | List + Create API | 10h | Task 00 ✅ | GET list, POST create, GET dot-ke-khai-hop-le | ⏳ |
| Task 02 | 02 | Detail + Chốt + Sửa + Xóa | 10h | Task 00 ✅ | GET detail, POST chốt, PUT sửa, DELETE xóa, GET hồ sơ lẻ | ⏳ |
| Task 03 | 03 | Export Service | 8h | Task 00 ✅ | exportC17Pdf (DOCX→PDF), exportChiTietXlsx (15 cột) | ⏳ |

---

## Dependency Graph

```
Task 00 (Data Model Foundation)
    ├─→ Task 01 (List + Create API)
    ├─→ Task 02 (Detail + Chốt + Sửa + Xóa)
    └─→ Task 03 (Export Service)
```

**Task 01, 02, 03 có thể phát triển song song sau khi Task 00 hoàn thành.**

---

## Sync Points

### Shared (Task 00)
- **Entities**: `NvDoiSoatThuLao`, `NvDoiSoatThuLaoHoSo`, `NvChinhSachThuLao` (read-only)
- **Repositories**: `DoiSoatThuLaoRepository`, `DoiSoatThuLaoHoSoRepository`
- **Enums**: `TrangThaiDoiSoat` (DANG_DOI_SOAT, DA_DOI_SOAT)
- **Error codes**: range `3200xx` — DOI_SOAT_NOT_FOUND, DOI_SOAT_TEN_DUPLICATE, DOI_SOAT_INVALID_STATE, DOI_SOAT_FORBIDDEN, DOI_SOAT_EMPTY_DOT
- **Permission**: `DOI_SOAT_THU_LAO` in `PermissionConstants`
- **State Machine**: `DoiSoatThuLaoStateMachine`

### Services shared
- Task 01 uses: `DoiSoatThuLaoRepository.existsByTenDoiSoat()`, `HoSoRepository.findAllByDotKeKhaiIdIn()`
- Task 02 uses: `DoiSoatThuLaoHoSoRepository.deleteByDoiSoatThuLaoIdAndHoSoIdIn()`
- Task 03 uses: `DoiSoatThuLaoRepository.findById()`, `DoiSoatThuLaoHoSoRepository.findAllByDoiSoatThuLaoId()`

### No Events
- Feature không dùng async events — tất cả Synchronous

---

## Files

| File | Task | Description |
|------|------|-------------|
| [task_00_data_model_solution.md](task_00_data_model_solution.md) | 00 | Entities, Enums, Repos, State Machine, Error Codes |
| [task_01_list_create_solution.md](task_01_list_create_solution.md) | 01 | GET list, POST create, GET dot-ke-khai-hop-le |
| [task_02_detail_chot_sua_xoa_solution.md](task_02_detail_chot_sua_xoa_solution.md) | 02 | GET detail, POST chốt, PUT sửa, DELETE xóa, GET hồ sơ lẻ |
| [task_03_export_solution.md](task_03_export_solution.md) | 03 | exportC17Pdf (PDF), exportChiTietXlsx (XLSX) |

---

## Progress

- [ ] Task 00: Data Model Foundation (0/7 sections)
- [ ] Task 01: List + Create API (0/10 sections)
- [ ] Task 02: Detail + Chốt + Sửa + Xóa (0/10 sections)
- [ ] Task 03: Export Service (0/8 sections)

---

## Notes

- FK to category tables: `dm_*` dùng `ma` (NOT `id`) — xem AGENTS.md
- `phuongAnDong`, `phuongThucDong`: dùng **Java key-value map** load-once per query, không JOIN (vì source từ nhiều bảng dm khác nhau tùy loại nghiệp vụ)
- UNIQUE KEY `uk_ho_so_reconciliation (ho_so_id)` — DB enforce 1 hồ sơ chỉ thuộc 1 kỳ
- Soft delete: `deleted_at IS NULL` filter qua `@SQLDelete` + `@Where`
- All code must be actual Java, not pseudocode
