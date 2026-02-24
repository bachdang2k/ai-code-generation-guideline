# Phương án Kỹ thuật — Đối Soát Thù Lao (Portal)

> **Phiên bản**: 1.0  
> **Ngày tạo**: 2026-02-23  
> **Tham chiếu URD**: US-TL01 → US-TL05 — `vivas_bhxh_nghiepvu/Portal quản trị/Đối soát/Đối soát thù lao/`  
> **Tham chiếu Checklist QC**: `docs/tasks/checklist_qc_portal_doisoat_thulao.md`  
> **DB Schema**: `src/main/resources/db/migration/V5__script_doisoat.sql`

---

## 1. Overview & Scope

**Feature Summary**: Đối soát thù lao | Type: New Feature | Module: Portal | Complexity: Medium

**Business Context**
- User Stories: US-TL01, US-TL02, US-TL03, US-TL04, US-TL05
- Actors: Admin Đại lý thu (all agencies), Đại lý thu (own-created only)
- Business Value: Tổng hợp và xác nhận thù lao thu hộ từ các đợt kê khai đã được CQ BHXH xử lý; xuất biểu mẫu C17-TS để đối chiếu chính thức với cơ quan BHXH

**Technical Context**
- Service Class: `DoiSoatThuLaoService`
- Primary Entities: `NvDoiSoatThuLao`, `NvDoiSoatThuLaoHoSo`, `NvChinhSachThuLao`
- State Machine: Yes (`DANG_DOI_SOAT` → `DA_DOI_SOAT`)

**Data Enrichment Context**
- Required External Data: Không có external API call
- Required Internal Data:
  - `nv_dot_ke_khai` — đợt kê khai (trạng thái, ngày gửi, đại lý)
  - `nv_ho_so_dang_ky` — hồ sơ (trạng thái, số tiền, biên lai)
  - `nv_chinh_sach_thu_lao` — tỷ lệ thù lao (tra theo vùng + loại NV + phương thức + phương án)
  - `dm_donvi_hanhchinh` — mã vùng (I/II/III) theo tỉnh/TP nơi cư trú
  - `agency` — thông tin đại lý thu
- Enrichment Strategy: Synchronous (snapshot tại thời điểm tạo kỳ đối soát)
- Calculation Ownership: `DoiSoatThuLaoService` owns all commission rate lookups and calculations

**Scope**
- In Scope:
  - CRUD kỳ đối soát thù lao (TL01–TL05)
  - Tính toán thù lao theo `nv_chinh_sach_thu_lao`
  - Xuất file **C17-TS** (PDF từ template `.docs`)
  - Xuất file **DS chi tiết hồ sơ** (XLSX)
  - Chốt kỳ đối soát (immutable sau khi chốt)
  - RBAC: Admin (all) vs Đại lý thu (own `created_by_user_id`)
- Out of Scope:
  - Cập nhật tỷ lệ thù lao qua UI (seed data cố định)
  - Đối soát hàng ngày (`nv_doi_soat_hang_ngay` — feature riêng biệt)
  - Thanh toán/Giao dịch tài chính (handled by `QuanLyThuService`)

**URD References**: Primary: `vivas_bhxh_nghiepvu/Portal quản trị/Đối soát/Đối soát thù lao/`, Related: `context/context.md`, `V5__script_doisoat.sql`

**API Prefix**: `/api/portal/v1/doi-soat-thu-lao`

---

## 2. Acceptance Criteria Mapping

### Scenario → Endpoint Mapping

| Scenario | API Endpoint | Method | Purpose |
|----------|-------------|--------|---------|
| US-TL01, S1.1–1.5 | `/api/portal/v1/doi-soat-thu-lao` | GET | Danh sách kỳ đối soát |
| US-TL02, S2.1–2.4 | `/api/portal/v1/doi-soat-thu-lao` | POST | Tạo kỳ đối soát |
| US-TL03, S3.1 | `/api/portal/v1/doi-soat-thu-lao/{id}` | GET | Xem chi tiết |
| US-TL03, S3.2 | `/api/portal/v1/doi-soat-thu-lao/{id}/chot` | POST | Chốt đối soát |
| US-TL03, S3.3 | `/api/portal/v1/doi-soat-thu-lao/{id}/export-c17` | GET | Xuất C17-TS (PDF) |
| US-TL03, S3.4 | `/api/portal/v1/doi-soat-thu-lao/{id}/export-chi-tiet` | GET | Xuất DS chi tiết (XLSX) |
| US-TL04, S4.1–4.3 | `/api/portal/v1/doi-soat-thu-lao/{id}` | PUT | Sửa kỳ đối soát |
| US-TL04, S4.4 | `/api/portal/v1/doi-soat-thu-lao/{id}/ho-so-le` | GET | Danh sách hồ sơ lẻ có thể thêm |
| US-TL05, S5.1 | `/api/portal/v1/doi-soat-thu-lao/{id}` | DELETE | Xóa kỳ đối soát |
| US-TL02 sidebar | `/api/portal/v1/doi-soat-thu-lao/dot-ke-khai-hop-le` | GET | Đợt kê khai hợp lệ khi tạo |

### Scenario → Validation Mapping

| Scenario | Validation Rule | Implementation |
|----------|----------------|----------------|
| TL02 | Tên đối soát: bắt buộc, unique | NotBlank + unique constraint DB |
| TL02 | Giai đoạn: bắt buộc, không chọn ngày tương lai | NotNull + validate denNgay <= today |
| TL02 | Đại lý thu: bắt buộc | NotNull |
| TL02 | CQ BHXH: bắt buộc | NotNull |
| TL02 | Tối thiểu 1 đợt kê khai | validate dotKeKhaiIds.size >= 1 |
| TL03/04/05 | Chỉ thao tác kỳ `DANG_DOI_SOAT` | Check trangThai trước khi thực hiện |
| TL04/05 | RBAC owner check | createdByUserId == currentUserId OR isAdmin |

### Scenario → Data Enrichment Mapping

| Scenario | Data Source | Enrichment Type | Sync/Async | Purpose |
|----------|-------------|-----------------|------------|---------|
| TL02 — Tạo kỳ | `nv_ho_so_dang_ky.ti_le_thu_lao` (cột có sẵn) | Snapshot thù lao | Sync (at aproved) | Đọc `ti_le_thu_lao` NULL → tính `thu_lao_duoc_huong` cho từng hồ sơ (Logic already implemented in TinhTiLeThuLaoEventListener)|
| TL03 — Chi tiết | `nv_doi_soat_thu_lao_ho_so` | Read denormalized | Sync | Đọc snapshot đã lưu (soTienDong, tiLeThuLao, thuLaoDuocHuong) |
| TL03 — Xuất C17 | `nv_doi_soat_thu_lao_ho_so` + agency config | Report generation | Sync | PDF từ template |
| TL03 — Xuất XLSX | `nv_doi_soat_thu_lao_ho_so` | Report generation | Sync | XLSX danh sách hồ sơ |

### Scenario → Business Logic Mapping

| Scenario | Business Rule | Implementation Scope |
|----------|--------------|---------------------|
| TL02 | Lọc đợt kê khai hợp lệ | Query với điều kiện status + ngày gửi + đại lý + còn hồ sơ chưa đối soát |
| TL02 | Include đợt cũ tồn đọng | OR condition: ngày gửi < tuNgay AND chưa đối soát |
| TL02/04 | 1 hồ sơ chỉ thuộc 1 kỳ | UNIQUE KEY `uk_ho_so_reconciliation (ho_so_id)` DB-enforced |
| TL03 | Tính thù lao | `thu_lao_duoc_huong = so_tien_dong × ti_le_thu_lao` |
| TL03/Chốt | State transition | DANG_DOI_SOAT → DA_DOI_SOAT |
| TL05/Xóa | Revert hồ sơ | DELETE `nv_doi_soat_thu_lao_ho_so` records → hồ sơ available again |

---

## 3. Architecture & Data Flow

### System Architecture

```
[Portal Client]
    → [DoiSoatThuLaoController]
    → [DoiSoatThuLaoService]
        ├── [DoiSoatThuLaoRepository]    → nv_doi_soat_thu_lao
        ├── [DoiSoatThuLaoHoSoRepository] → nv_doi_soat_thu_lao_ho_so
        ├── [HoSoRepository]             → nv_ho_so_dang_ky (đọc ti_le_thu_lao, so_tien_dong)
        ├── [DotKeKhaiRepository]        → nv_dot_ke_khai
        └── [ExportService]              → C17-TS PDF / XLSX
    // Note: ChinhSachThuLaoRepository và DonViHanhChinhRepository KHÔNG cần
    // ti_le_thu_lao đã có sẵn trong nv_ho_so_dang_ky.ti_le_thu_lao
```

### Request Flow — Tạo kỳ đối soát (TL02)

1. Controller nhận `TaoDoiSoatThuLaoRequest`
2. Service validate input (tên unique, giai đoạn, đại lý, CQ BHXH, ≥1 đợt)
3. Service validate RBAC (Admin or Đại lý thu)
4. Service load danh sách `NvDotKeKhai` từ `dotKeKhaiIds`
5. Service load danh sách hồ sơ từ `nv_ho_so_dang_ky` WHERE `dot_ke_khai_id IN :dotKeKhaiIds`
6. For each hồ sơ: đọc `ti_le_thu_lao` (cột NULL trong `nv_ho_so_dang_ky`) → tính `thu_lao_duoc_huong`
7. Service tạo `NvDoiSoatThuLao` + list `NvDoiSoatThuLaoHoSo` (snapshot: soTienDong, tiLeThuLao, thuLaoDuocHuong)
8. Service persist (atomic transaction)
9. Log: `log.info("DoiSoat created: id={}, daiLyId={}, soHoSo={}", id, daiLyId, soHoSo)`
10. Return `DoiSoatThuLaoDetailResponse`

### Request Flow — Xuất C17-TS (TL03)

1. Controller nhận GET `/{id}/export-c17`
2. Service load `NvDoiSoatThuLao` + `NvDoiSoatThuLaoHoSo` list
3. Service validate RBAC
4. Service validate trangThai == DANG_DOI_SOAT
5. Service load agency representative name từ DB config
6. Service gom nhóm biên lai, sắp xếp ngày biên lai tăng dần
7. Service fill template `template_doi_soat_C17_TS.doc` bằng `DocxTemplateService`
8. Service convert → PDF → trả về ResponseEntity với `Content-Disposition: attachment`

### State Machine — Kỳ đối soát

```
[Tạo mới] ──────────────► DANG_DOI_SOAT
                                │
              ┌─────────────────┼──────────────────┐
              │                 │                  │
          [Sửa]           [Chốt]             [Xóa]
              │                 │                  │
          DANG_DOI_SOAT    DA_DOI_SOAT        [Deleted + revert hồ sơ]
```

**Hard Rules**:
- `DA_DOI_SOAT` là trạng thái cuối — không thể sửa/xóa/chốt lại
- Xóa: chỉ `DANG_DOI_SOAT`, soft delete `deleted_at` + xóa `nv_doi_soat_thu_lao_ho_so`
- 1 hồ sơ đã có trong `nv_doi_soat_thu_lao_ho_so` → không thể thêm vào kỳ khác (UNIQUE KEY DB)

---

## 4. API Design

### API Endpoints

| Method | Path | Purpose | Auth | Ref |
|--------|------|---------|------|-----|
| GET | `/api/portal/v1/doi-soat-thu-lao` | Danh sách kỳ đối soát | @PreAuthorize | US-TL01 |
| POST | `/api/portal/v1/doi-soat-thu-lao` | Tạo kỳ đối soát | @PreAuthorize | US-TL02 |
| GET | `/api/portal/v1/doi-soat-thu-lao/{id}` | Xem chi tiết | @PreAuthorize | US-TL03 |
| POST | `/api/portal/v1/doi-soat-thu-lao/{id}/chot` | Chốt kỳ đối soát | @PreAuthorize | US-TL03 |
| GET | `/api/portal/v1/doi-soat-thu-lao/{id}/export-c17` | Xuất C17-TS PDF | @PreAuthorize | US-TL03 |
| GET | `/api/portal/v1/doi-soat-thu-lao/{id}/export-chi-tiet` | Xuất DS chi tiết XLSX | @PreAuthorize | US-TL03 |
| PUT | `/api/portal/v1/doi-soat-thu-lao/{id}` | Sửa kỳ đối soát | @PreAuthorize | US-TL04 |
| GET | `/api/portal/v1/doi-soat-thu-lao/{id}/ho-so-le` | Hồ sơ lẻ có thể thêm | @PreAuthorize | US-TL04 |
| DELETE | `/api/portal/v1/doi-soat-thu-lao/{id}` | Xóa kỳ đối soát | @PreAuthorize | US-TL05 |
| GET | `/api/portal/v1/doi-soat-thu-lao/dot-ke-khai-hop-le` | Đợt kê khai hợp lệ | @PreAuthorize | US-TL02 |

### Request DTOs (Pseudocode)

**TaoDoiSoatThuLaoRequest**:
- tenDoiSoat: String (required, unique)
- daiLyThuId: Integer (required)
- coQuanBhxhId: Integer (required)
- tuNgay: LocalDate (required, <= today)
- denNgay: LocalDate (required, <= today, >= tuNgay)
- dotKeKhaiIds: List\<Integer\> (required, min size = 1)

**SuaDoiSoatThuLaoRequest**:
- themHoSoIds: List\<Integer\> (nullable — hồ sơ lẻ cần thêm)
- xoaHoSoIds: List\<Integer\> (nullable — hồ sơ cần xóa khỏi kỳ)

**DotKeKhaiHopLeRequest** (query params):
- daiLyThuId: Integer (required)
- tuNgay: LocalDate (required)
- denNgay: LocalDate (required)

**HoSoLeRequest** (query params):
- keyword: String (nullable — tìm theo mã hồ sơ, tên người tham gia)
- page, size: Integer

### Response DTOs (Pseudocode)

**DoiSoatThuLaoListItemResponse**:
- id: Integer
- tenDoiSoat: String
- tuNgay: LocalDate
- denNgay: LocalDate
- tenDaiLyThu: String
- tenCoQuanBhxh: String
- soLuongHoSo: Integer
- tongTien: BigDecimal
- thuLaoDuocHuong: BigDecimal
- trangThai: TrangThaiDoiSoat (enum: DANG_DOI_SOAT / DA_DOI_SOAT)

**DoiSoatThuLaoDetailResponse**:
- id: Integer
- tenDoiSoat: String
- tuNgay: LocalDate
- denNgay: LocalDate
- tenDaiLyThu: String
- tenCoQuanBhxh: String
- trangThai: TrangThaiDoiSoat
- hoSoList: Page\<DoiSoatHoSoDetailItem\>
- tongSoTienDong: BigDecimal (total all pages)
- tongThuLao: BigDecimal (total all pages)

**DoiSoatHoSoDetailItem** (dòng trong bảng chi tiết — đọc từ snapshot `nv_doi_soat_thu_lao_ho_so`):
- id: Integer
- maHoSo: String  ← `nv_doi_soat_thu_lao_ho_so.ma_ho_so` (denorm)
- tenNguoiThamGia: String  ← `nv_doi_soat_thu_lao_ho_so.ten_nguoi_tham_gia` (denorm)
- thuTuc: String (BHXH Tự nguyện / BHYT Hộ gia đình)  ← `loai_nghiep_vu` (denorm)
- tinhTp: String  ← JOIN `dm_donvi_hanhchinh` tại query time
- vung: String (I / II / III)  ← `nv_doi_soat_thu_lao_ho_so.vung` (denorm)
- phuongAnDong: String  ← `nv_doi_soat_thu_lao_ho_so.phuong_an_dong` (denorm)
- phuongThucDong: String  ← `nv_doi_soat_thu_lao_ho_so.phuong_thuc_dong` (denorm)
- soBienLai: String  ← `nv_doi_soat_thu_lao_ho_so.so_bien_lai` (denorm)
- soTienDong: BigDecimal  ← **`nv_doi_soat_thu_lao_ho_so.so_tien_dong`** (snapshot từ `nv_ho_so_dang_ky` lúc tạo kỳ)
- tiLeThuLao: BigDecimal (%)  ← **`nv_doi_soat_thu_lao_ho_so.ti_le_thu_lao`** (snapshot từ `nv_ho_so_dang_ky.ti_le_thu_lao` lúc tạo kỳ)
- thuLaoDuocHuong: BigDecimal  ← `nv_doi_soat_thu_lao_ho_so.thu_lao_duoc_huong` (= soTienDong × tiLeThuLao)
- tenDaiLyThuHo: String  ← JOIN `agency` tại query time
- tenNhanVienThu: String  ← `nv_doi_soat_thu_lao_ho_so` metadata (denorm)

**DotKeKhaiHopLeResponse** (item):
- dotKeKhaiId: Integer
- tenDotKeKhai: String
- thuTuc: String (602 / 603)
- soLuongHoSoChuaDoiSoat: Integer

**HoSoLeResponse** (item):
- hoSoId: Integer
- maHoSo: String
- tenDotKeKhai: String
- tenNguoiThamGia: String
- thuTuc: String
- trangThai: String

### FK Resolution Strategy (from AGENTS.md)

> **Rule**: `dm_*` tables dùng `ma` (code) làm FK, **KHÔNG phải `id`**. Business tables lưu `ma` để JOIN. Khi display cần `ten` (name), phải JOIN — không lazy load từng entity.

| Field trong Response | Cột lưu trữ (code) | Resolution Strategy | N+1 Risk |
|---------------------|--------------------|--------------------|----------|
| `tenDaiLyThu` | `nv_doi_soat_thu_lao.dai_ly_thu_id` → `agency.id` | JOIN `agency` ON `a.id = ds.dai_ly_thu_id` trong list query | ✅ JOIN 1 lần |
| `tenCoQuanBhxh` | `nv_doi_soat_thu_lao.co_quan_bhxh_id` → `dm_co_quan_bhxh.id` | JOIN `dm_co_quan_bhxh` ON `cq.id = ds.co_quan_bhxh_id` trong list query | ✅ JOIN 1 lần |
| `vung` trong detail | `nv_doi_soat_thu_lao_ho_so.vung` lưu code I/II/III (display as-is) | Direct read — không cần resolve | ✅ |
| `phuongAnDong` | `nv_doi_soat_thu_lao_ho_so.phuong_an_dong` lưu code `ma` | ⚠️ **Key-value map** — xem note bên dưới | ✅ Nếu dùng map |
| `phuongThucDong` | `nv_doi_soat_thu_lao_ho_so.phuong_thuc_dong` lưu code `ma` | ⚠️ **Key-value map** — xem note bên dưới | ✅ Nếu dùng map |
| `soTienDong` | `nv_doi_soat_thu_lao_ho_so.so_tien_dong` snapshot | Direct read — no JOIN needed | ✅ |
| `tiLeThuLao` | `nv_doi_soat_thu_lao_ho_so.ti_le_thu_lao` snapshot | Direct read — no JOIN needed | ✅ |

> **⚠️ Note — `phuongAnDong` và `phuongThucDong` (dành cho task_solution)**
> 
> Dữ liệu có thể đến từ **nhiều bảng `dm_*` khác nhau** (mixed source) tùy theo `loai_nghiep_vu`, nên JOIN đơn giản lên 1 bảng không khả thi.
> 
> **Giải pháp**: Dùng **Java key-value map** (load 1 lần khi start query, cache trong service scope):  
> - Load toàn bộ `Map<String, String>` từ tất cả bảng dm liên quan → `{"ma" → "ten"}`  
> - Apply map khi build response DTO (O(1) per row, không N+1)  
> - Implementation detail: task_solution (không define ở PAKT)


### Validation Matrix

| Field | Validation | Error Code | Message |
|-------|------------|------------|---------|
| tenDoiSoat | Required, unique | DOI_SOAT_TEN_DUPLICATE | "Tên đối soát đã tồn tại" |
| tuNgay | Required, <= today | INVALID_DATE_RANGE | "Giai đoạn đối soát không hợp lệ" |
| denNgay | Required, <= today, >= tuNgay | INVALID_DATE_RANGE | "Giai đoạn đối soát không hợp lệ" |
| dotKeKhaiIds | Required, min 1 | DOI_SOAT_EMPTY_DOT | "Vui lòng chọn tối thiểu 1 đợt kê khai." |
| trangThai (chốt/sửa/xóa) | Must be DANG_DOI_SOAT | DOI_SOAT_INVALID_STATE | "Kỳ đối soát đã chốt, không thể thực hiện thao tác này" |
| Ownership (sửa/xóa/chốt) | Admin OR createdByUserId | DOI_SOAT_FORBIDDEN | "Bạn không có quyền thực hiện thao tác này" |

### Query Logic

#### Query: GetDoiSoatList
**Purpose**: Danh sách kỳ đối soát với filter
**Tables**: `nv_doi_soat_thu_lao` (ds) LEFT JOIN `agency` (a ON a.id = ds.dai_ly_thu_id) LEFT JOIN `dm_co_quan_bhxh` (cq ON cq.id = ds.co_quan_bhxh_id)
**Joins**: Fetch `tenDaiLyThu = a.ten`, `tenCoQuanBhxh = cq.ten` trong cùng 1 query (tránh N+1)
**Sub-query aggregation** (tránh N+1 cho COUNT/SUM):
```
  LEFT JOIN (
    SELECT doi_soat_thu_lao_id,
           COUNT(*) AS so_luong_ho_so,
           SUM(so_tien_dong) AS tong_tien,
           SUM(thu_lao_duoc_huong) AS tong_thu_lao
    FROM nv_doi_soat_thu_lao_ho_so
    GROUP BY doi_soat_thu_lao_id
  ) agg ON agg.doi_soat_thu_lao_id = ds.id
```
**Scope**: WHERE ds.deleted_at IS NULL AND (:daiLyThuId IS NULL OR ds.dai_ly_thu_id = :daiLyThuId — Admin only) AND (:trangThai IS NULL OR ds.trang_thai = :trangThai) AND (:tenSearch IS NULL OR ds.ten_doi_soat LIKE %) AND (:tuNgay IS NULL OR ds.tu_ngay >= :tuNgay) AND (:denNgay IS NULL OR ds.den_ngay <= :denNgay) AND (Đại lý thu: ds.created_by_user_id = :currentUserId)
**N+1 Risk**: ✅ NONE — tất cả aggregation dùng GROUP BY sub-query, không load từng entity riêng

#### Query: GetDotKeKhaiHopLe
**Purpose**: Đợt kê khai hợp lệ để tạo kỳ đối soát
**Tables**: `nv_dot_ke_khai` (dkk) LEFT JOIN `nv_ho_so_dang_ky` (hs ON hs.dot_ke_khai_id = dkk.id) LEFT JOIN `nv_doi_soat_thu_lao_ho_so` (dshs ON dshs.ho_so_id = hs.id)
**Scope**:
- dkk.dai_ly_thu_id = :daiLyThuId
- dkk.trang_thai IN ('CQ_BHXH_DANG_XU_LY', 'THANH_CONG')
- (dkk.ngay_gui BETWEEN :tuNgay AND :denNgay) OR (dkk.ngay_gui < :tuNgay AND soHoSoChuaDoiSoat > 0)
- soHoSoChuaDoiSoat = COUNT(hs.id) WHERE dshs.ho_so_id IS NULL
  (JOIN `nv_doi_soat_thu_lao_ho_so` chỉ qua `ho_so_id`, không join qua `dot_ke_khai_id`)
**Group By**: dkk.id
**Order**: dkk.ngay_gui ASC

#### Query: GetHoSoLe
**Purpose**: Hồ sơ lẻ có thể thêm vào kỳ đang sửa
**Tables**: `nv_ho_so_dang_ky` (hs) LEFT JOIN `nv_nguoi_tham_gia` (ntt ON ntt.ho_so_id = hs.id) LEFT JOIN `nv_dot_ke_khai` (dkk ON dkk.id = hs.dot_ke_khai_id)
**Scope**: hs.trang_thai IN ('THANH_CONG', 'CQ_BHXH_DANG_XU_LY') AND hs.id NOT IN (SELECT ho_so_id FROM nv_doi_soat_thu_lao_ho_so) AND hs.id NOT IN (current kỳ's ho_so_ids)
**N+1 Risk**: ✅ Dùng NOT IN sub-query 1 lần — không loop từng hồ sơ

---

## 5. Data Model & State

### Entity → Table Mapping

| Entity | Table | PK | Status | Key Fields |
|--------|-------|-----|--------|-----------|
| NvDoiSoatThuLao | nv_doi_soat_thu_lao | id | **Existing** (V5) | tenDoiSoat (UNIQUE), trangThai, createdByUserId, deletedAt |
| NvDoiSoatThuLaoHoSo | nv_doi_soat_thu_lao_ho_so | id | **Existing** (V5) | doiSoatThuLaoId, hoSoId (UNIQUE), soTienDong, tiLeThuLao, thuLaoDuocHuong |
| NvChinhSachThuLao | nv_chinh_sach_thu_lao | id | **Existing** (V5) | loaiNghiepVu, loaiSanPham, phuongThucDong, vungDvhc, phuongAnDong, tiLeThuLao |
| DmDonViHanhChinh | dm_donvi_hanhchinh | id | **Existing** (Updated V5) | maVung (I/II/III), cap='tinh' |
| NvDotKeKhai | nv_dot_ke_khai | id | **Existing** | trangThai, ngayGui, daiLyThuId |

### Entity Definitions (Scope)

**NvDoiSoatThuLao** (đọc từ V5):
- Fields: id, tenDoiSoat (unique), tuNgay, denNgay, daiLyThuId, coQuanBhxhId, trangThai (DANG_DOI_SOAT/DA_DOI_SOAT), nguoiChotDoiSoat, ngayChotDoiSoat, deletedAt, createdAt, updatedAt, createdByUserId
- Annotations: @Entity, @Table, @Enumerated, @CreatedDate, @LastModifiedDate, @SQLDelete (soft delete), @Where(deletedAt IS NULL)

**NvDoiSoatThuLaoHoSo** (đọc từ V5):
- Fields: id, doiSoatThuLaoId, hoSoId (UNIQUE), dotKeKhaiId, soTienDong, tiLeThuLao, thuLaoDuocHuong, maHoSo (denorm), tenNguoiThamGia (denorm), loaiNghiepVu (denorm), loaiSanPham (denorm), vung (denorm), phuongAnDong (denorm), phuongThucDong (denorm), soBienLai (denorm), ngayTao
- Note: Snapshot data — điền khi tạo kỳ đối soát, không thay đổi sau

**NvChinhSachThuLao** (đọc từ V5):
- Fields: id, loaiNghiepVu, loaiSanPham, phuongThucDong, vungDvhc, phuongAnDong, trangThai (1/0), tiLeThuLao (DECIMAL 5,2), ghiChu

### Cross-Entity Enrichment Mapping

| Primary Entity | Enriched From | Relationship | Fields Added | Timing | N+1 Risk |
|----------------|---------------|--------------|-------------|--------|-----------|
| NvDoiSoatThuLaoHoSo | NvHoSoDangKy | JOIN at creation (batch) | soTienDong, tiLeThuLao, maHoSo, soBienLai, loaiNghiepVu | At kỳ creation | ✅ Batch load, không per-row |
| NvDoiSoatThuLaoHoSo | Mixed `dm_*` tables | **Key-value map** (Java load-once per query, không JOIN) | phuongAnDong ten, phuongThucDong ten | Per list/detail query | ✅ Map lookup O(1) per row |

> **FK Rule (AGENTS.md + Note)**: `phuong_an_dong` và `phuong_thuc_dong` lưu code `ma` nhưng source từ **nhiều bảng `dm_*` khác nhau** tùy `loai_nghiep_vu`. JOIN đơn lên 1 bảng không khả thi. Dùng **Java key-value map** — implementation detail trong task_solution.

### State Transitions

| Current | Event | Next | Validation |
|---------|-------|------|-----------|
| None | Tạo kỳ | DANG_DOI_SOAT | validate input + đợt hợp lệ |
| DANG_DOI_SOAT | Sửa | DANG_DOI_SOAT | ownership + status check |
| DANG_DOI_SOAT | Chốt | DA_DOI_SOAT | ownership + status check |
| DANG_DOI_SOAT | Xóa | deleted | ownership + status check + cascade delete ho_so records |
| DA_DOI_SOAT | Any | ❌ BLOCKED | BusinessException(DOI_SOAT_INVALID_STATE) |

---

## 6. Business Logic Scope

### Service Method Structure (Pseudocode)

**METHOD taoDoiSoatThuLao(TaoDoiSoatThuLaoRequest) RETURNS DoiSoatThuLaoDetailResponse**
```
1. Validate JWT + get currentUser
2. Validate RBAC: Admin OR Đại lý thu
3. Validate tenDoiSoat unique (check DB)
4. Validate denNgay <= today, tuNgay <= denNgay
5. Validate dotKeKhaiIds.size >= 1
6. Load NvDotKeKhai list → validate status IN (CQ_BHXH_DANG_XU_LY, THANH_CONG)
7. Load hồ sơ từ các đợt kê khai (JOIN nv_ho_so_dang_ky WHERE dot_ke_khai_id IN :dotKeKhaiIds)
   // BATCH load toàn bộ — KHÔNG loop từng dotKeKhaiId (N+1)
8. FOR EACH hoSo:
   a. tiLeThuLao = hoSo.tiLeThuLao  // Đọc trực tiếp từ nv_ho_so_dang_ky.ti_le_thu_lao (NULL)
   b. IF tiLeThuLao IS NULL: tiLeThuLao = BigDecimal.ZERO  // Không có tỷ lệ → 0%
   c. soTienDong = hoSo.soTienDong  // Đọc từ nv_ho_so_dang_ky (cột phù hợp — so_tien_phai_dong hoặc mapping)
   d. thuLaoDuocHuong = soTienDong × tiLeThuLao / 100
   e. Build NvDoiSoatThuLaoHoSo (snapshot: soTienDong, tiLeThuLao, thuLaoDuocHuong + denorm fields)
9. Build NvDoiSoatThuLao (status=DANG_DOI_SOAT, createdByUserId=currentUser.id)
10. Persist @Transactional: save NvDoiSoatThuLao → saveAll(hoSoList) [batch 500]
11. log.info("DoiSoat created: id={}, daiLyId={}, dotKeKhaiCount={}, hoSoCount={}", ...)
12. Return mapped response
```

**METHOD chotDoiSoatThuLao(Integer id) RETURNS void**
```
1. Load NvDoiSoatThuLao by id (throw NOT_FOUND if absent or deleted)
2. Validate RBAC: isAdmin OR doiSoat.createdByUserId == currentUser.id
3. Validate trangThai == DANG_DOI_SOAT (else throw DOI_SOAT_INVALID_STATE)
4. Set trangThai = DA_DOI_SOAT
5. Set nguoiChotDoiSoat = currentUser.id, ngayChotDoiSoat = now()
6. Save @Transactional
7. log.info("DoiSoat chot: id={}, by={}", id, currentUserId)
```

**METHOD xoaDoiSoatThuLao(Integer id) RETURNS void**
```
1. Load NvDoiSoatThuLao by id
2. Validate RBAC
3. Validate trangThai == DANG_DOI_SOAT
4. @Transactional:
   a. DELETE all NvDoiSoatThuLaoHoSo WHERE doiSoatThuLaoId = id  (revert hồ sơ)
   b. Soft delete: doiSoat.deletedAt = now()
   c. Save
5. log.info("DoiSoat deleted: id={}, hoSoReverted={}", id, count)
```

**METHOD suaDoiSoatThuLao(Integer id, SuaDoiSoatThuLaoRequest) RETURNS DoiSoatThuLaoDetailResponse**
```
1. Load + RBAC + state validate
2. IF xoaHoSoIds not empty:
   DELETE NvDoiSoatThuLaoHoSo WHERE doiSoatThuLaoId=id AND hoSoId IN :xoaHoSoIds
   // Batch DELETE — không loop từng id (N+1)
3. IF themHoSoIds not empty:
   // BATCH load tất cả hồ sơ 1 lần: WHERE id IN :themHoSoIds (không per-hoSo query)
   hoSoList = hoSoRepository.findAllById(themHoSoIds)
   FOR EACH hoSo in hoSoList:
     - validate chưa thuộc kỳ nào (SELECT 1 WHERE ho_so_id=? — chạy sau batch load)
     - validate trangThai IN ('THANH_CONG', 'CQ_BHXH_DANG_XU_LY')
     - tính tiLeThuLao từ hoSo.tiLeThuLao (nv_ho_so_dang_ky.ti_le_thu_lao), NULL → 0
     - thuLaoDuocHuong = soTienDong × tiLeThuLao / 100
     - Build NvDoiSoatThuLaoHoSo snapshot
   saveAll(newHoSoList) [batch 500]
4. Reload và trả về updated DoiSoatThuLaoDetailResponse
```

### N+1 Prevention Summary

| Scenario | Risk Point | Giải pháp |
|----------|-----------|----------|
| Danh sách kỳ đối soát | COUNT/SUM hồ sơ mỗi kỳ | GROUP BY sub-query trong 1 query |
| Chi tiết — bảng hồ sơ | Resolve `phuongAnDong`, `phuongThucDong` mỗi dòng | **Key-value map** (Java, load once — impl trong task_solution) |
| Tạo kỳ — load hồ sơ từ nhiều đợt | Loop từng đợt kê khai để query hồ sơ | `WHERE dot_ke_khai_id IN :dotKeKhaiIds` (1 batch query) |
| Sửa — thêm hồ sơ lẻ | `findById` từng hoSoId | `findAllById(List<Integer>)` (1 batch query) |
| Sửa — validate "đã thuộc kỳ" | Check từng hoSo riêng lẻ | `WHERE ho_so_id IN :ids` (batch) |
| Xuất XLSX — load toàn bộ hồ sơ | Lazy load entity per row | Projection query + key-value map cho dm fields (impl trong task_solution) |
| GetDotKeKhaiHopLe — đếm hồ sơ chưa đối soát | Per-đợt count | GROUP BY dkk.id trong 1 query với LEFT JOIN |


### Nguồn dữ liệu tính thù lao

> **Thiết kế**: `ti_le_thu_lao` được tính sẵn và lưu vào cột `nv_ho_so_dang_ky.ti_le_thu_lao` (DECIMAL(5,2) NULL) khi hồ sơ được xử lý (bởi upstream flow). Feature Đối soát thù lao **chỉ đọc** cột này — không tự tra cứu `nv_chinh_sach_thu_lao`.

```
FUNCTION getTiLeThuLaoForSnapshot(NvHoSoDangKy hoSo) RETURNS BigDecimal
  tiLeThuLao = hoSo.tiLeThuLao            // FROM nv_ho_so_dang_ky.ti_le_thu_lao (NULL)
  IF tiLeThuLao IS NULL:
    RETURN BigDecimal.ZERO                // Hồ sơ không có tỷ lệ → 0% (BHYT không áp dụng, etc.)
  RETURN tiLeThuLao

FUNCTION calcThuLaoDuocHuong(BigDecimal soTienDong, BigDecimal tiLeThuLao) RETURNS BigDecimal
  RETURN soTienDong × tiLeThuLao / 100   // Đơn vị: tiLeThuLao là %, VD: 13.50 → ×0.1350
```

### Xuất C17-TS (PDF)

```
FUNCTION exportC17(doiSoatId) RETURNS byte[]
  1. Load NvDoiSoatThuLao + NvDoiSoatThuLaoHoSo list
  2. Load agency representative name từ DB config (NOT current user)
  3. Group biên lai theo so_bien_lai, sắp xếp ngày ASC
  4. Build C17DataModel:
     - (1): tenCoQuanBhxhTinh
     - (2): để trống
     - (3): tenDaiLyThu  
     - (4): soThuTuVanBan (auto-generated)
     - (5): ngayTaoKy (formatted dd/MM/yyyy)
     - Bảng biên lai rows: STT, quyenSo="", so=soBienLai, ngay=ngayBienLai,
                           bhxh=(soTien nếu BHXH else null), bhyt=(soTien nếu BHYT else null),
                           tongSo=soTien
     - (6): tongSoToBienLai (count)
     - (7): tongSoTienNop (SUM tongSo)
     - (8): SoTienBangChu(tongSoTienNop)
     - (9): tenDaiDienDaiLyThu (from DB config)
     - (10): để trống
  5. Fill template_doi_soat_C17_TS.doc using DocxTemplateService
  6. Convert DOCX → PDF
  7. Return byte[] with filename="C17-TS.pdf"
```

### Xuất DS Chi Tiết Hồ Sơ (XLSX)

```
FUNCTION exportChiTiet(doiSoatId) RETURNS byte[]
  1. Load ALL NvDoiSoatThuLaoHoSo (no pagination)
  2. Build XLSX with 15 columns per row:
     STT | Đợt kê khai | Mã hồ sơ | Người tham gia | Thủ tục | Tỉnh/TP | Vùng |
     Phương án đóng | Phương thức đóng | Số biên lai | Số tiền đóng |
     Tỷ lệ thù lao (%) | Thù lao được hưởng | Đại lý thu hộ | Nhân viên thu
  3. Add footer row: TỔNG CỘNG | (blank×9) | SUM(soTienDong) | blank | SUM(thuLaoDuocHuong) | blank×2
  4. Return byte[] with filename="DS chi tiết hồ sơ.xlsx"
```

### Error Handling

| Error | Code | Exception | HTTP |
|-------|------|-----------|------|
| Kỳ đối soát không tồn tại | DOI_SOAT_NOT_FOUND | BusinessException | 404 |
| Tên đối soát đã tồn tại | DOI_SOAT_TEN_DUPLICATE | BusinessException | 200 |
| Giai đoạn không hợp lệ | INVALID_DATE_RANGE | BusinessException | 200 |
| Chưa chọn đợt kê khai | DOI_SOAT_EMPTY_DOT | BusinessException | 200 |
| Kỳ đã chốt, không thể thao tác | DOI_SOAT_INVALID_STATE | BusinessException | 200 |
| Không có quyền | DOI_SOAT_FORBIDDEN | BusinessException | 403 |

---

## 7. Integration & Enrichment Scope

### Internal Cross-Feature Enrichment

| Enrichment Source | Service/Entity | Method | Sync/Async | Error Handling |
|-------------------|----------------|--------|------------|----------------|
| Tỷ lệ thù lao | HoSoRepository | `nv_ho_so_dang_ky.ti_le_thu_lao` (cột NULL) | Sync at creation | NULL → BigDecimal.ZERO (0%) |
| Hồ sơ + soTienDong | HoSoRepository | findsByDotKeKhaiIdIn() | Sync at creation | Skip if empty |
| Agency representative | AgencyConfigRepository | findDaiDienByAgencyId() | Sync at export | Return blank if not configured |

### Export Services

**C17-TS PDF Export**:
- Service: `DoiSoatExportService`
- Template: `src/main/resources/templates/template_doi_soat_C17_TS.doc`
- Library: Apache POI (XWPF) for DOCX fill → Conversion to PDF
- Method: `byte[] exportC17Pdf(Integer doiSoatId)`

**DS Chi tiết XLSX Export**:
- Service: `DoiSoatExportService`
- Library: Apache POI (XSSF)
- Method: `byte[] exportChiTietXlsx(Integer doiSoatId)`
- Columns: 15 cột theo spec Scenario 3.4

---

## 8. Security Scope

### Authentication
All APIs: JWT token required — `AuthenticationService.getCurrentUserRequired()`

### Authorization Matrix

| API | Role | Condition | Data Scope |
|-----|------|-----------|-----------|
| GET list | Admin + Đại lý thu | Authenticated | Admin: tất cả; Đại lý thu: `created_by_user_id = currentUser.id` |
| POST create | Admin + Đại lý thu | Authenticated | Admin: bất kỳ đại lý; Đại lý thu: chỉ agency của mình |
| GET detail | Admin + Đại lý thu | Ownership check | Same as list |
| POST chot | Admin + Đại lý thu | Ownership + DANG_DOI_SOAT | Same |
| GET export | Admin + Đại lý thu | Ownership check | Same |
| PUT sua | Admin + Đại lý thu | Ownership + DANG_DOI_SOAT | Same |
| DELETE xoa | Admin + Đại lý thu | Ownership + DANG_DOI_SOAT | Same |

### Data Scoping (ABAC)

```
METHOD checkOwnership(doiSoat, currentUser) RETURNS void
  IF currentUser.isAdmin():
    RETURN  // Admin bypass
  IF doiSoat.createdByUserId != currentUser.id:
    THROW BusinessException(DOI_SOAT_FORBIDDEN)
```

**Admin filter cho list**:
```
IF currentUser.isAdmin():
  // Hiển thị bộ lọc Đại lý thu — filter by daiLyThuId (optional param)
ELSE:
  // Đại lý thu: ẩn bộ lọc, hardcode WHERE created_by_user_id = currentUser.id
```

### Security Checklist
- [ ] Portal APIs require JWT (`getCurrentUserRequired()`)
- [ ] `@PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.DOI_SOAT_THU_LAO)")`
- [ ] Ownership check trong service layer trước mọi write operation
- [ ] Soft delete không expose `deleted_at IS NOT NULL` records

---

## 9. Impact & Dependencies

### Performance Requirements

| Requirement | Metric | Implementation |
|------------|--------|----------------|
| GET list | < 1s | Index trên `dai_ly_thu_id`, `trang_thai`, `deleted_at` |
| GET chi tiết | < 2s | Index trên `doi_soat_thu_lao_id` (nv_doi_soat_thu_lao_ho_so) |
| Export XLSX/PDF | < 5s | Stream response, không load toàn bộ vào memory |
| Tính thù lao lúc tạo | < 3s | Batch lookup nv_chinh_sach_thu_lao (cache friendly) |

### DDL (Đã có trong V5 — không cần thêm)

**Các bảng đã sẵn sàng:**
- `nv_doi_soat_thu_lao` ✅
- `nv_doi_soat_thu_lao_ho_so` ✅ (UNIQUE KEY uk_ho_so_reconciliation)
- `nv_chinh_sach_thu_lao` ✅ + seed data đầy đủ
- `dm_donvi_hanhchinh.ma_vung` ✅ (I/II/III)
- `nv_ho_so_dang_ky.ti_le_thu_lao` ✅

**Indices đề xuất (nếu chưa có):**
```sql
-- Tăng tốc query danh sách kỳ đối soát
CREATE INDEX IF NOT EXISTS idx_doi_soat_dai_ly ON nv_doi_soat_thu_lao (dai_ly_thu_id, deleted_at);
CREATE INDEX IF NOT EXISTS idx_doi_soat_trang_thai ON nv_doi_soat_thu_lao (trang_thai, deleted_at);
CREATE INDEX IF NOT EXISTS idx_doi_soat_creator ON nv_doi_soat_thu_lao (created_by_user_id, deleted_at);

-- Tăng tốc đếm hồ sơ chưa đối soát
-- uk_ho_so_reconciliation (ho_so_id) đã có
```

### Compatibility

| Aspect | Impact | Mitigation |
|--------|--------|-----------|
| DB schema | Không thay đổi schema | Dùng V5 đã có |
| Existing APIs | Không breaking change | New endpoints |
| Permissions | Thêm permission mới | Cập nhật `PermissionConstants.DOI_SOAT_THU_LAO` |

### Risks

| Risk | Mitigation |
|------|-----------|
| UNIQUE constraint conflict khi 2 user cùng thêm hồ sơ | DB-level UNIQUE KEY enforce, bắt DataIntegrityViolationException → BusinessException |
| PDF generation chậm (template lớn) | Async generation + streaming response |
| Số tờ biên lai lớn trong C17 | Pagination bảng PDF + streaming |

---

## 10. Implementation Scope

### Code Patterns Reference
- **rule_technical.md**: §4.1 Exception Handling, §4.2 Logging, §4.3 Service Layer, §4.4 API Response, §4.6 Security
- **API prefix**: `/api/portal/v1/doi-soat-thu-lao`
- **Error codes**: Dùng `MessageResponseDict` enum, range: 3200xx

### Task Breakdown Checklist

**Foundation Tasks (Data Model)**
- [ ] Entity: `NvDoiSoatThuLao.java` — map `nv_doi_soat_thu_lao`, enum `TrangThaiDoiSoat`
- [ ] Entity: `NvDoiSoatThuLaoHoSo.java` — map `nv_doi_soat_thu_lao_ho_so`
- [ ] Enum: `TrangThaiDoiSoat` (DANG_DOI_SOAT, DA_DOI_SOAT)
- [ ] Repository: `DoiSoatThuLaoRepository`, `DoiSoatThuLaoHoSoRepository`
  - Note: `ChinhSachThuLaoRepository` **KHÔNG cần** — ti_le_thu_lao đọc từ `nv_ho_so_dang_ky`
- [ ] Error codes: `DOI_SOAT_NOT_FOUND`, `DOI_SOAT_TEN_DUPLICATE`, `DOI_SOAT_INVALID_STATE`, `DOI_SOAT_FORBIDDEN`, `DOI_SOAT_EMPTY_DOT` → thêm vào `MessageResponseDict`
- [ ] Permission: `DOI_SOAT_THU_LAO` → thêm vào `PermissionConstants`

**API Tasks (Controllers & DTOs)**
- [ ] Request DTOs: `TaoDoiSoatThuLaoRequest`, `SuaDoiSoatThuLaoRequest`
- [ ] Response DTOs: `DoiSoatThuLaoListItemResponse`, `DoiSoatThuLaoDetailResponse`, `DoiSoatHoSoDetailItem`, `DotKeKhaiHopLeResponse`, `HoSoLeResponse`
- [ ] Controller: `DoiSoatThuLaoController` — 10 endpoints
- [ ] Swagger: @Tag, @Operation, @PreAuthorize annotations

**Service Tasks (Business Logic)**
- [ ] Service: `DoiSoatThuLaoService`
  - [ ] `taoDoiSoatThuLao()` — với thù lao lookup + snapshot
  - [ ] `getDoiSoatList()` — với filter + RBAC scope
  - [ ] `getDoiSoatDetail()` — với phân trang bảng hồ sơ
  - [ ] `chotDoiSoatThuLao()` — state transition
  - [ ] `suaDoiSoatThuLao()` — thêm/xóa hồ sơ
  - [ ] `xoaDoiSoatThuLao()` — soft delete + revert
  - [ ] `getDotKeKhaiHopLe()` — điều kiện hợp lệ
  - [ ] `getHoSoLe()` — popup thêm hồ sơ lẻ
  - [ ] `getTiLeThuLaoForSnapshot()` — private helper: đọc `hoSo.tiLeThuLao` (NULL→0)

**Export Tasks**
- [ ] Service: `DoiSoatExportService`
  - [ ] `exportC17Pdf(Integer doiSoatId)` — fill template .doc → PDF
  - [ ] `exportChiTietXlsx(Integer doiSoatId)` — 15-column XLSX
- [ ] Dependency: Apache POI XWPF (DOCX) + PDF converter

**Testing Tasks**
- [ ] Unit tests: `DoiSoatThuLaoServiceTest` ≥ 80% coverage
- [ ] Unit tests: `DoiSoatExportServiceTest` — verify column headers + data sample
- [ ] Integration tests: `DoiSoatThuLaoRepositoryIT` (TestContainers MySQL)
- [ ] API tests: `DoiSoatThuLaoControllerTest` — happy paths + auth checks

### Data Enrichment Checklist
- [x] §1: Data Enrichment Context populated (snapshot strategy)
- [x] §2: Scenario → Enrichment Mapping complete
- [x] §3: Enrichment pattern = Synchronous snapshot at kỳ creation
- [x] §5: Cross-entity mapping documented (nv_doi_soat_thu_lao_ho_so stores snapshot)
- [x] §6: Domain ownership: DoiSoatThuLaoService owns all lookup + calc
- [x] §7: Export services documented
- [x] §9: DDL already in V5, no new migration needed

### Definition of Done

**✅ MANDATORY (100%)**
- [ ] US-TL01: Danh sách với filter + RBAC admin/đại lý thu
- [ ] US-TL02: Tạo kỳ + tính thù lao đúng + unique name
- [ ] US-TL03: Chi tiết + chốt + xuất C17 + xuất XLSX
- [ ] US-TL04: Sửa (thêm/xóa hồ sơ lẻ) + tính lại tổng
- [ ] US-TL05: Xóa + revert hồ sơ con
- [ ] State transitions đúng (DANG → DA / Deleted)
- [ ] RBAC: Admin bypass / Đại lý thu ownership check
- [ ] Error codes trong MessageResponseDict
- [ ] Unit tests ≥ 80% coverage
- [ ] Export C17-TS PDF đúng mapping 10 trường + bảng biên lai
- [ ] Export XLSX đúng 15 cột + dòng tổng cộng
- [ ] log.info() đúng format cho mọi write operation

**⚙️ ENHANCED**
- [ ] Error messages tiếng Việt user-friendly
- [ ] Swagger docs đầy đủ

**🎯 BONUS**
- [ ] Edge case: UNIQUE constraint violation → friendly error
- [ ] Performance: Export XLSX streaming (> 1000 hồ sơ)
