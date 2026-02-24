# 🧠 Brainstorming – Đối Soát Thù Lao

> **Phase**: Phase 1 – Brainstorming (BHXH Backend Workflow)  
> **Feature**: Đối soát thù lao (Commission Reconciliation)  
> **URDs**: US-TL01 → US-TL05  
> **Date**: 2026-02-23  
> **Status**: ✅ Understanding Lock CONFIRMED — Design Validated

---

## ✅ Understanding Lock — Đã Xác Nhận

### Tóm tắt feature
Module **Đối soát thù lao** cho phép Admin/Đại lý thu:
- Tạo **kỳ đối soát** từ các đợt kê khai đủ điều kiện
- Tính **thù lao** cho từng hồ sơ theo `nv_chinh_sach_thu_lao`
- Xuất **C17-TS** (PDF) + **DS chi tiết hồ sơ** (XLSX)
- **Chốt** kỳ đối soát để xác nhận số liệu với CQ BHXH

### Actors & RBAC (Q4 confirmed)
| Actor | Quyền |
|-------|-------|
| **Admin** | Tạo/Sửa/Xóa/Chốt bất kỳ kỳ đối soát nào |
| **Đại lý thu** | Tạo/Sửa/Xóa/Chốt — **chỉ kỳ do chính mình tạo** (`created_by_user_id`) |

### Confirmed Answers

| # | Câu hỏi | Kết quả |
|---|---------|---------|
| Q1 | Nguồn tỷ lệ thù lao | Bảng `nv_chinh_sach_thu_lao` — seed data cố định trong DB |
| Q2 | Xác định Vùng | Tỉnh/TP nơi cư trú người tham gia (`dm_donvi_hanhchinh.ma_vung`) |
| Q3 | Hồ sơ "đã đối soát" khi nào? | **Chỉ khi kỳ chứa nó được Chốt** (`DA_DOI_SOAT`) |
| Q4 | RBAC | Đại lý thu: toàn quyền nhưng **phải là người tạo** |

---

## 🏗️ DB Schema Hiện Có (V5__script_doisoat.sql)

Schema đã được thiết kế sẵn, không cần tạo mới:

### `nv_chinh_sach_thu_lao`
Bảng seed data tỷ lệ thù lao theo: `loai_nghiep_vu`, `loai_san_pham`, `phuong_thuc_dong`, `vung_dvhc`, `phuong_an_dong`.

### `nv_doi_soat_thu_lao` (kỳ đối soát)
| Cột | Mô tả |
|-----|-------|
| `ten_doi_soat` | UNIQUE name |
| `tu_ngay` / `den_ngay` | Giai đoạn đối soát |
| `dai_ly_thu_id` | FK → đại lý |
| `co_quan_bhxh_id` | FK → CQ BHXH |
| `trang_thai` | `DANG_DOI_SOAT` / `DA_DOI_SOAT` |
| `created_by_user_id` | User tạo (dùng cho RBAC Đại lý thu) |
| `nguoi_chot_doi_soat` + `ngay_chot_doi_soat` | Audit chốt |
| `deleted_at` | Soft delete |

### `nv_doi_soat_thu_lao_ho_so` (chi tiết hồ sơ)
| Cột | Mô tả |
|-----|-------|
| `doi_soat_thu_lao_id` | FK → kỳ đối soát |
| `ho_so_id` | FK → hồ sơ (UNIQUE → 1 hồ sơ chỉ thuộc 1 kỳ, nhưng theo Q3 chỉ khi chốt) |
| `so_tien_dong` / `ti_le_thu_lao` / `thu_lao_duoc_huong` | Snapshot tính toán |
| Các cột denormalized | `ma_ho_so`, `ten_nguoi_tham_gia`, `vung`, `so_bien_lai`, v.v. |

> [!IMPORTANT]
> **Lưu ý Q3 ↔ DB constraint**: `UNIQUE KEY uk_ho_so_reconciliation (ho_so_id)` trong `nv_doi_soat_thu_lao_ho_so` đang **enforce rằng 1 hồ sơ chỉ thuộc 1 kỳ đối soát bất kể trạng thái**. Điều này **mâu thuẫn** với Q3 (chỉ khóa khi Chốt). Cần quyết định xử lý (xem Decision Log #1).

---

## 5️⃣ Design Approaches

### Option A — UNIQUE constraint giữ nguyên (Recommended ✅)
**Hồ sơ bị "chiếm" ngay khi thêm vào kỳ** — kể cả kỳ `DANG_DOI_SOAT`.

- `UNIQUE KEY uk_ho_so_reconciliation (ho_so_id)` trong DB giữ nguyên
- Logic "số hồ sơ chưa đối soát" = hồ sơ chưa có record nào trong `nv_doi_soat_thu_lao_ho_so`
- Khi xóa kỳ đối soát → **xóa** các record trong `nv_doi_soat_thu_lao_ho_so` → hồ sơ "thả" ra
- ✅ Simple, DB enforce nhất quán, không có race condition
- ⚠️ Trade-off: Nếu kỳ `DANG_DOI_SOAT` bị xóa → hồ sơ available trở lại (OK với US-TL05)

### Option B — Chỉ khóa khi Chốt
- Bỏ UNIQUE constraint, thay bằng điều kiện: `WHERE doi_soat.trang_thai = 'DA_DOI_SOAT'`
- "Hồ sơ chưa đối soát" = không có record trong kỳ đã **chốt** (`DA_DOI_SOAT`)
- ⚠️ Race condition: Cùng lúc 2 kỳ đang `DANG_DOI_SOAT` có thể chứa cùng hồ sơ
- ❌ Phức tạp hơn, khó enforce tại DB level

### Kết luận: Chọn **Option A** — đơn giản nhất, DB-enforced, phù hợp với URD thực tế

---

## 6️⃣ Thiết Kế Hệ Thống

### State Machine – Kỳ đối soát
```
  Tạo mới
     ↓
DANG_DOI_SOAT ──[Sửa]──→ DANG_DOI_SOAT
      │
  [Chốt]──→ DA_DOI_SOAT  (locked, read-only)
      │
  [Xóa]──→ deleted_at = NOW()  (soft delete + revert hồ sơ)
```

### API Endpoints (Portal – CMS)

```
# TL01 - Danh sách
GET    /api/cms/v1/doi-soat-thu-lao
       ?page, size, search, tuNgay, denNgay, daiLyThuId, trangThai

# TL02 - Tạo
POST   /api/cms/v1/doi-soat-thu-lao

# TL03 - Chi tiết + Chốt + Xuất
GET    /api/cms/v1/doi-soat-thu-lao/{id}
POST   /api/cms/v1/doi-soat-thu-lao/{id}/chot
GET    /api/cms/v1/doi-soat-thu-lao/{id}/export-c17        (PDF)
GET    /api/cms/v1/doi-soat-thu-lao/{id}/export-chi-tiet   (XLSX)

# TL04 - Sửa
PUT    /api/cms/v1/doi-soat-thu-lao/{id}
GET    /api/cms/v1/doi-soat-thu-lao/{id}/ho-so-le          (danh sách có thể thêm)
POST   /api/cms/v1/doi-soat-thu-lao/{id}/ho-so             (thêm hồ sơ lẻ)
DELETE /api/cms/v1/doi-soat-thu-lao/{id}/ho-so/{hoSoId}    (xóa hồ sơ khỏi kỳ)

# TL05 - Xóa
DELETE /api/cms/v1/doi-soat-thu-lao/{id}

# TL02 - Sidebar: Đợt kê khai hợp lệ
GET    /api/cms/v1/doi-soat-thu-lao/dot-ke-khai-hop-le
       ?daiLyThuId, tuNgay, denNgay
```

### Business Logic – Tính Tỷ Lệ Thù Lao

Tra cứu `nv_chinh_sach_thu_lao` theo:
1. `loai_nghiep_vu` = loại hồ sơ (BHXH/BHYT)
2. `loai_san_pham` = loại sản phẩm từ hồ sơ
3. `phuong_thuc_dong` = phương thức đóng của hồ sơ
4. `vung_dvhc` = `ma_vung` từ `dm_donvi_hanhchinh` theo tỉnh/TP nơi cư trú
5. `phuong_an_dong` = TM (tham mới) hoặc TT (tiếp tục)
6. `trang_thai = 1` (có hiệu lực)

Nếu không tìm thấy → `ti_le_thu_lao = 0%` (BHYT không có tỷ lệ).

### Business Logic – Điều Kiện Đợt Kê Khai Hợp Lệ (TL02)

```sql
-- Đợt kê khai hợp lệ khi tạo kỳ đối soát
WHERE dot_ke_khai.trang_thai IN ('CQ_BHXH_DANG_XU_LY', 'THANH_CONG')
  AND dot_ke_khai.dai_ly_thu_id = :daiLyThuId
  AND (
    -- Điều kiện 1: Ngày gửi trong giai đoạn
    (dot_ke_khai.ngay_gui BETWEEN :tuNgay AND :denNgay)
    OR
    -- Điều kiện 2: Đợt cũ còn hồ sơ tồn đọng (chưa đối soát)
    (dot_ke_khai.ngay_gui < :tuNgay AND so_luong_ho_so_chua_doi_soat > 0)
  )
  AND so_luong_ho_so_chua_doi_soat > 0

-- "Hồ sơ chưa đối soát" = hồ sơ chưa có trong nv_doi_soat_thu_lao_ho_so
```

### RBAC Authorization
```java
// @PreAuthorize trong Controller
// Admin: tất cả
// Đại lý thu: phải là created_by_user_id
if (!isAdmin && doiSoat.getCreatedByUserId() != currentUserId) {
    throw new BusinessException(FORBIDDEN);
}
```

---

## 7️⃣ Decision Log

| # | Quyết định | Lý do | Thay thế |
|---|-----------|-------|----------|
| 1 | **Chọn Option A**: UNIQUE KEY `(ho_so_id)` → hồ sơ bị khóa ngay khi thêm vào kỳ bất kể trạng thái | DB-level enforcement, tránh race condition, phù hợp với schema V5 đã thiết kế | Option B: chỉ khóa khi Chốt — phức tạp hơn, race condition |
| 2 | **ti_le_thu_lao = 0%** khi không tìm thấy trong `nv_chinh_sach_thu_lao` | URD: "BHYT không có tỷ lệ, mặc định 0%" | Throw exception — quá restrictive |
| 3 | **Soft delete** kỳ đối soát (`deleted_at`) + xóa record `nv_doi_soat_thu_lao_ho_so` | Revert hồ sơ về chưa đối soát, giữ audit trail | Hard delete — mất dữ liệu audit |
| 4 | **Denormalize** thông tin hồ sơ vào `nv_doi_soat_thu_lao_ho_so` | Snapshot tại thời điểm đối soát, trích xuất C17 chính xác dù hồ sơ gốc bị sửa | Join trực tiếp — số liệu thay đổi khi hồ sơ bị sửa |
| 5 | Scope Đại lý thu: chỉ thao tác kỳ **do mình tạo** (`created_by_user_id`) | User requirement Q4 | Scope theo `dai_ly_thu_id` của user |

---

## ⏩ Next Steps

Sau khi xác nhận design, tiến hành **Phase 2: Acceptance Criteria Checklist** theo workflow:
- Tạo file: `docs/tasks/checklist_qc_portal_doisoat_thulao.md`
