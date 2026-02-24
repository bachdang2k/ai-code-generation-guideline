# CHECKLIST QC — Portal: Đối Soát Thù Lao

> **Version**: 1.0  
> **Rule**: rule_create_checklist.md v5.0  
> **Feature**: Đối soát thù lao (Commission Reconciliation)  
> **URDs**: US-TL01 → US-TL05  
> **Created**: 2026-02-23  
> **Status**: Draft

---

## Section 1: Feature Overview

### Feature: Đối Soát Thù Lao (Portal)

#### Scope
- **User Stories**: US-TL01, US-TL02, US-TL03, US-TL04, US-TL05
- **Actors**:
  - **Admin Đại lý thu**: Quản lý tất cả kỳ đối soát của mọi đại lý
  - **Đại lý thu**: Quản lý kỳ đối soát do chính mình tạo (`created_by_user_id`)
- **Workflow**: Đại lý thu chọn đợt kê khai đủ điều kiện → Hệ thống tổng hợp hồ sơ và tính thù lao → Xem chi tiết / Xuất C17-TS → Chốt đối soát để xác nhận với CQ BHXH

#### URD References
- **Primary URDs**:
  - `vivas_bhxh_nghiepvu/Portal quản trị/Đối soát/Đối soát thù lao/[US-TL01] Xem danh sách đối soát thù lao.md`
  - `vivas_bhxh_nghiepvu/Portal quản trị/Đối soát/Đối soát thù lao/[US-TL02] Tạo đối soát thù lao.md`
  - `vivas_bhxh_nghiepvu/Portal quản trị/Đối soát/Đối soát thù lao/[US-TL03] Xem chi tiết đối soát.md`
  - `vivas_bhxh_nghiepvu/Portal quản trị/Đối soát/Đối soát thù lao/[US-TL04] Sửa đối soát thù lao.md`
  - `vivas_bhxh_nghiepvu/Portal quản trị/Đối soát/Đối soát thù lao/[US-TL05] Xóa đối soát thù lao.md`
- **Related**: `bhxh-java-backend/context/context.md`, `V5__script_doisoat.sql`

#### Business Context
- **Feature Type**: New feature — Đối soát thù lao thu hộ với CQ BHXH
- **Business Value**: Tổng hợp và xác nhận thù lao cho đại lý thu hộ dựa trên hồ sơ đã được CQ BHXH xử lý; xuất C17-TS để đối chiếu chính thức

---

## Section 2: Happy Path Scenarios ⭐ MANDATORY (100% Required)

---

### User Story: US-TL01 — Xem danh sách đối soát thù lao

#### Scenario 1.1: Admin xem toàn bộ danh sách cùng bộ lọc đại lý
- [ ] Given: User đăng nhập với role **Admin**
- [ ] And: Hệ thống có nhiều kỳ đối soát của nhiều đại lý
- [ ] When: User truy cập màn hình Danh sách đối soát thù lao
- [ ] Then: Hệ thống hiển thị bảng với các cột: STT, Tên đối soát, Giai đoạn đối soát, Tên đại lý thu, Cơ quan BHXH chi trả thù lao, Số lượng hồ sơ, Tổng tiền, Thù lao được hưởng, Trạng thái
- [ ] And: Bộ lọc **Đại lý thu** (Dropdown) hiển thị — vì role = Admin
- [ ] And: Mặc định phân trang 20 dòng/trang

#### Scenario 1.2: Đại lý thu xem danh sách — không có bộ lọc đại lý
- [ ] Given: User đăng nhập với role **Đại lý thu**
- [ ] When: User truy cập màn hình Danh sách đối soát thù lao
- [ ] Then: Hệ thống chỉ hiển thị các kỳ đối soát do chính user tạo
- [ ] And: Bộ lọc **Đại lý thu** **không** hiển thị (role ≠ Admin)

#### Scenario 1.3: Lọc theo giai đoạn đối soát và trạng thái
- [ ] Given: User ở màn hình Danh sách đối soát thù lao
- [ ] When: User chọn **Giai đoạn đối soát** = 01/01/2026 - 31/01/2026 và **Trạng thái** = Đang đối soát → bấm Tìm kiếm
- [ ] Then: Hệ thống chỉ hiển thị các kỳ đối soát có ngày trong khoảng đó và trạng thái phù hợp

#### Scenario 1.4: Tìm kiếm theo tên đối soát
- [ ] Given: User ở màn hình Danh sách
- [ ] When: User nhập tên đối soát vào ô tìm kiếm → bấm Tìm kiếm
- [ ] Then: Hệ thống lọc và hiển thị các kỳ đối soát có tên chứa chuỗi tìm kiếm

#### Scenario 1.5: Menu thao tác hiển thị đúng theo trạng thái
- [ ] Given: Bảng danh sách có cả kỳ `Đang đối soát` và `Đã đối soát`
- [ ] When: User mở **Menu Thao tác** của kỳ `Đang đối soát`
- [ ] Then: Menu hiển thị: **Xem chi tiết**, **Sửa đối soát**, **Chốt đối soát**, **Xóa đối soát**
- [ ] When: User mở **Menu Thao tác** của kỳ `Đã đối soát`
- [ ] Then: Menu chỉ hiển thị: **Xem chi tiết**

---

### User Story: US-TL02 — Tạo đối soát thù lao

#### Scenario 2.1: Admin tạo kỳ đối soát thành công
- [ ] Given: User đăng nhập với role **Admin**
- [ ] And: Có đợt kê khai hợp lệ (trạng thái `CQ BHXH đang xử lý` hoặc `Thành công`, còn hồ sơ chưa đối soát)
- [ ] When: User bấm **Tạo đối soát** → Nhập tên đối soát (unique) → Chọn Đại lý thu → Chọn Cơ quan BHXH → Chọn Giai đoạn đối soát → Tích chọn ít nhất 1 đợt kê khai → Bấm **Tạo đối soát**
- [ ] Then: Kỳ đối soát được tạo với trạng thái `Đang đối soát`
- [ ] And: Hệ thống điều hướng user sang màn hình **Xem chi tiết đối soát**
- [ ] And: Toast thành công: **"Tạo đối soát thù lao thành công."**

#### Scenario 2.2: Đại lý thu tạo kỳ đối soát — Tên đại lý thu tự điền và disable
- [ ] Given: User đăng nhập với role **Đại lý thu**
- [ ] When: User mở popup Tạo đối soát
- [ ] Then: Trường **Tên đại lý thu** hiển thị tên của chính đại lý đó và bị **disable** (không cho sửa)
- [ ] And: Trường **Cơ quan BHXH chi trả thù lao** hiển thị CQ BHXH mặc định gắn với đại lý đó

#### Scenario 2.3: Danh sách đợt kê khai tải lại khi thay đổi giai đoạn
- [ ] Given: User đang trong popup Tạo đối soát sau khi chọn Đại lý thu
- [ ] When: User thay đổi **Giai đoạn đối soát**
- [ ] Then: Danh sách đợt kê khai bên dưới **tự động tải lại** (Reload) theo giai đoạn mới
- [ ] And: Hệ thống tự động include thêm các đợt kê khai **cũ hơn giai đoạn** nhưng còn hồ sơ chưa đối soát

#### Scenario 2.4: Kỳ đối soát tổng hợp đúng hồ sơ sau khi tạo
- [ ] Given: User tạo kỳ đối soát và chọn đợt kê khai A (10 hồ sơ) và đợt kê khai B (5 hồ sơ)
- [ ] When: Bấm Tạo đối soát
- [ ] Then: Kỳ đối soát chứa tổng 15 hồ sơ từ 2 đợt
- [ ] And: Các hồ sơ trong đợt A và B không thể được thêm vào kỳ đối soát khác (bị UNIQUE constraint)

---

### User Story: US-TL03 — Xem chi tiết đối soát

#### Scenario 3.1: Xem danh sách hồ sơ với tính toán thù lao
- [ ] Given: Kỳ đối soát đang ở trạng thái `Đang đối soát`
- [ ] When: User truy cập chi tiết kỳ đối soát
- [ ] Then: Phần **Thông tin đối soát** hiển thị: Tên đối soát, Giai đoạn đối soát, Tên đại lý thu, Cơ quan BHXH chi trả thù lao
- [ ] And: Bảng danh sách hồ sơ gồm các cột: STT, Mã hồ sơ, Người tham gia, Thủ tục, Tỉnh/TP, Vùng, Phương án đóng, Số biên lai, Số tiền đóng, Tỷ lệ thù lao (%), Thù lao được hưởng, Đại lý thu hộ, Nhân viên thu
- [ ] And: **Thù lao được hưởng** của mỗi hồ sơ = `Số tiền đóng × Tỷ lệ thù lao (%)`
- [ ] And: Dòng **Tổng cộng** cuối bảng hiển thị tổng Số tiền đóng và tổng Thù lao được hưởng (in đậm, màu đỏ)
- [ ] And: Cột STT, Mã hồ sơ được pinned cố định khi scroll ngang

#### Scenario 3.2: Chốt đối soát thành công
- [ ] Given: Kỳ đối soát ở trạng thái `Đang đối soát`
- [ ] When: User bấm **Chốt đối soát** → Popup xác nhận xuất hiện → User bấm **Xác nhận**
- [ ] Then: Trạng thái kỳ đối soát chuyển sang `Đã đối soát`
- [ ] And: Các nút Sửa, Chốt đối soát, Xuất C17 bị ẩn đi
- [ ] And: Toast thành công: **"Chốt đối soát thù lao thành công."**

#### Scenario 3.3: Xuất C17-TS (PDF)
- [ ] Given: Kỳ đối soát ở trạng thái `Đang đối soát`
- [ ] When: User bấm **Xuất C17**
- [ ] Then: Hệ thống tạo và tải xuống file **C17-TS.pdf** (tên file: `C17-TS`)
- [ ] And: Tiêu đề tài liệu: **"ĐỐI CHIẾU BIÊN LAI THU TIỀN ĐÓNG BHXH TỰ NGUYỆN, BHYT"**
- [ ] And: File được sinh đúng theo mẫu `template_doi_soat_C17_TS.doc` với mapping các trường như sau:

  **Thông tin header (góc trái trên):**

  | Ký hiệu | Nội dung điền | Nguồn dữ liệu |
  |---------|--------------|---------------|
  | **(1)** | Tên CQ BHXH tỉnh/thành phố (cấp trên) | `co_quan_bhxh` tỉnh gắn với kỳ đối soát |
  | **(2)** | Tên CQ BHXH huyện/quận (cấp dưới) | Để trống |
  | **(3)** | Tên Điểm thu / Đại lý thu | `dai_ly_thu` gắn với kỳ đối soát |
  | **(4)** | Số văn bản (VD: `01`, `02`...) | Số thứ tự tự sinh, tăng dần từ `01` |
  | **(5)** | Ngày lập văn bản (dd/mm/yyyy) | `created_at` của kỳ đối soát thù lao |

  **Bảng danh sách biên lai** — sắp xếp theo ngày biên lai từ **cũ nhất → mới nhất**:

  > Cấu trúc bảng gồm **7 cột** theo template thực (2 nhóm header):
  > - Nhóm **"Số biên lai"**: Quyển số | Số | Ngày
  > - Nhóm **"Số tiền thu"**: BHXH | BHYT | Tổng số

  | Cột # | Tên cột | Nội dung | Điều kiện |
  |-------|---------|---------|-----------|
  | STT | STT | Số thứ tự tăng dần, bắt đầu từ `01` | Luôn điền |
  | (1) | Quyển số | Để trống | — |
  | (2) | Số | Số biên lai (`so_bien_lai`) | Luôn điền |
  | (3) | Ngày | Ngày ghi trên biên lai (dd/mm/yyyy) | Luôn điền |
  | (4) | BHXH | Số tiền biên lai | **Chỉ điền** nếu biên lai thuộc loại BHXH tự nguyện; để trống nếu là BHYT |
  | (5) | BHYT | Số tiền biên lai | **Chỉ điền** nếu biên lai thuộc loại BHYT hộ gia đình; để trống nếu là BHXH |
  | (6) | Tổng số | = BHXH(4) + BHYT(5) của dòng đó | Luôn điền · Công thức: **7 = 5 + 6** |

  **Thông tin footer (dòng cuối bảng + ký tên):**

  | Ký hiệu | Nội dung điền | Nguồn dữ liệu |
  |---------|--------------|---------------|
  | **(6)** | Tổng số tờ biên lai kèm theo (VD: `..10..Tờ`) | Đếm tổng số dòng bảng biên lai |
  | **(7)** | Tổng số tiền nộp (VND, định dạng số) | Tổng cột **Tổng số** của tất cả biên lai |
  | **(8)** | Số tiền bằng chữ (tiếng Việt) | Chuyển đổi từ giá trị **(7)** |
  | **(9)** | Chữ ký + họ tên đại diện Điểm thu / Đại lý thu | Lấy từ cấu hình DB — **không phải user đang đăng nhập** |
  | **(10)** | Ô ký tên Phòng/Tổ Quản lý Thu của CQ BHXH | Để trống (phía CQ BHXH tự điền) |

- [ ] And: Thứ tự sắp xếp bảng biên lai: **ngày biên lai tăng dần** (cũ → mới)
- [ ] And: Một biên lai BHXH → cột (4) BHXH điền số tiền, cột (5) BHYT để trống — và ngược lại
- [ ] And: Cột Tổng số **(6)** = BHXH(4) + BHYT(5) của từng dòng (luôn bằng giá trị cột có dữ liệu)
- [ ] And: Số tiền **(7)** và chữ **(8)** phải nhất quán với nhau (cùng một giá trị)

#### Scenario 3.4: Tải file DS chi tiết hồ sơ (XLSX)
- [ ] Given: User đang ở màn hình chi tiết kỳ đối soát (bất kể trạng thái `Đang đối soát` hay `Đã đối soát`)
- [ ] When: User click vào hyperlink **Tải file chi tiết hồ sơ**
- [ ] Then: Hệ thống tạo và tải xuống file **"DS chi tiết hồ sơ.xlsx"**
- [ ] And: File gồm **15 cột** theo đúng thứ tự:

  | # | Tên cột | Mô tả |
  |---|---------|-------|
  | 1 | STT | Số thứ tự tăng dần, bắt đầu từ 1 |
  | 2 | Đợt kê khai | Tên/mã đợt kê khai (VD: `02/2025`) |
  | 3 | Mã hồ sơ | Mã hồ sơ duy nhất (VD: `BHXH-2025-010`) |
  | 4 | Người tham gia | Họ và tên người tham gia |
  | 5 | Thủ tục | Loại nghiệp vụ (VD: `BHXH Tự nguyện`) |
  | 6 | Tỉnh/TP | Tỉnh/TP nơi cư trú người tham gia |
  | 7 | Vùng | Vùng DVHC tương ứng (`I` / `II` / `III`) |
  | 8 | Phương án đóng | Phương án đóng của hồ sơ (VD: `Tăng mới`) |
  | 9 | Phương thức đóng | Phương thức đóng (VD: `3 tháng`) |
  | 10 | Số biên lai | Số biên lai của hồ sơ (VD: `000945`) |
  | 11 | Số tiền đóng | Số tiền đóng (định dạng VND có dấu phân cách, VD: `1.200.000`) |
  | 12 | Tỷ lệ thù lao (%) | Tỷ lệ thù lao tra từ danh mục (VD: `13,50%`) |
  | 13 | Thù lao được hưởng | = Số tiền đóng × Tỷ lệ thù lao (VD: `162.000`) |
  | 14 | Đại lý thu hộ | Tên đại lý thu hộ |
  | 15 | Nhân viên thu | Tên nhân viên thu thực hiện hồ sơ |

- [ ] And: Dữ liệu mẫu (ví dụ 2 dòng đầu):

  | STT | Đợt kê khai | Mã hồ sơ | Người tham gia | Thủ tục | Tỉnh/TP | Vùng | Phương án đóng | Phương thức đóng | Số biên lai | Số tiền đóng | Tỷ lệ thù lao (%) | Thù lao được hưởng | Đại lý thu hộ | Nhân viên thu |
  |-----|-------------|-----------|----------------|---------|---------|------|----------------|-------------------|-------------|-------------|-------------------|---------------------|--------------|--------------|
  | 1 | 02/2025 | BHXH-2025-010 | Trần Minh Chiến | BHXH Tự nguyện | TP.Hà Nội | I | Tăng mới | 3 tháng | 000945 | 1.200.000 | 13,50% | 162.000 | Media Hà Nội | Trần Vân Anh |
  | 2 | 02/2025 | BHXH-2025-009 | Trần Minh B | BHXH Tự nguyện | TP.Hà Nội | I | Tăng mới | 3 tháng | 000945 | 1.200.000 | 13,50% | 162.000 | Media Hà Nội | Trần Vân Anh |

- [ ] And: Cuối file có **dòng Tổng cộng**: tổng cột **Số tiền đóng** và tổng cột **Thù lao được hưởng** của tất cả hồ sơ
- [ ] And: File bao gồm **toàn bộ hồ sơ** trong kỳ (không phân trang, không giới hạn)

---

### User Story: US-TL04 — Sửa đối soát thù lao

#### Scenario 4.1: Thêm hồ sơ lẻ vào kỳ đối soát
- [ ] Given: Kỳ đối soát ở trạng thái `Đang đối soát`
- [ ] When: User bấm **Sửa** → Bấm **Thêm hồ sơ lẻ** → Tích chọn hồ sơ từ popup → Bấm **Xác nhận** → Bấm **Lưu**
- [ ] Then: Hồ sơ được thêm vào kỳ đối soát
- [ ] And: Dòng Tổng cộng được tính lại (Số tiền đóng + Thù lao)
- [ ] And: Toast thành công: **"Cập nhật đối soát thù lao thành công."**

#### Scenario 4.2: Xóa hồ sơ khỏi kỳ đối soát
- [ ] Given: User đang ở màn hình Sửa, kỳ đối soát `Đang đối soát`
- [ ] When: User bấm icon **Xóa** tại một hồ sơ trong bảng → Bấm **Lưu**
- [ ] Then: Hồ sơ bị loại khỏi kỳ đối soát
- [ ] And: Toast: **"Đã loại bỏ hồ sơ khỏi kỳ đối soát."**
- [ ] And: Dòng Tổng cộng được tính lại

#### Scenario 4.3: Tìm kiếm hồ sơ lẻ trong popup Thêm hồ sơ lẻ
- [ ] Given: Popup Thêm hồ sơ lẻ đang mở
- [ ] When: User nhập mã hồ sơ hoặc tên người tham gia vào ô tìm kiếm
- [ ] Then: Danh sách hồ sơ trong popup lọc theo từ khóa nhập vào

---

### User Story: US-TL05 — Xóa đối soát thù lao

#### Scenario 5.1: Xóa kỳ đối soát thành công
- [ ] Given: Kỳ đối soát ở trạng thái `Đang đối soát`
- [ ] When: User chọn **Xóa đối soát** từ Menu Thao tác → Popup xác nhận xuất hiện → User bấm **Đồng ý**
- [ ] Then: Kỳ đối soát bị xóa và không còn hiển thị trong danh sách
- [ ] And: Tất cả hồ sơ con thuộc kỳ đó được **revert** về trạng thái chưa đối soát (có thể thêm vào kỳ khác)
- [ ] And: Toast thành công: **"Xóa đối soát thù lao thành công."**

---

## Section 3: Common Use Scenarios ⭐ MANDATORY (50-70% Code Branch Coverage)

### User Story: US-TL01 — Xem danh sách

#### Scenario 1.6: Không tìm thấy kết quả khi tìm kiếm
- [ ] Given: User nhập từ khóa không khớp bất kỳ kỳ đối soát nào
- [ ] When: Bấm Tìm kiếm
- [ ] Then: Hệ thống hiển thị thông báo: **"Không tìm thấy kết quả phù hợp."**

#### Scenario 1.7: Lọc chỉ theo trạng thái Đã đối soát
- [ ] Given: User chọn Trạng thái = `Đã đối soát` trong bộ lọc
- [ ] When: Bấm Tìm kiếm
- [ ] Then: Danh sách chỉ hiển thị kỳ đã chốt; menu thao tác mỗi dòng chỉ có **Xem chi tiết**

---

### User Story: US-TL02 — Tạo đối soát

#### Scenario 2.5: Admin chọn đại lý thu → CQ BHXH mặc định thay đổi
- [ ] Given: Admin đang tạo đối soát
- [ ] When: Admin chọn **Tên đại lý thu** = Đại lý X
- [ ] Then: Trường **Cơ quan BHXH chi trả thù lao** tự động điền CQ BHXH gắn với Đại lý X
- [ ] And: Trường CQ BHXH chỉ enable sau khi đã chọn đại lý thu

#### Scenario 2.6: Đợt kê khai cũ tồn đọng được tự động include
- [ ] Given: Có đợt kê khai ngày gửi = 01/12/2025 (trước giai đoạn đối soát 01/01/2026 - 31/01/2026), còn 3 hồ sơ chưa đối soát
- [ ] When: User chọn giai đoạn đối soát = 01/01/2026 - 31/01/2026
- [ ] Then: Đợt kê khai cũ đó vẫn xuất hiện trong danh sách (tự động include tồn đọng)

#### Scenario 2.7: Tìm kiếm trong danh sách đợt kê khai
- [ ] Given: Danh sách đợt kê khai trong popup Tạo đối soát đang hiển thị
- [ ] When: User nhập từ khóa vào Searchbox
- [ ] Then: Danh sách lọc tương đối theo tên đợt kê khai

#### Scenario 2.8: Hồ sơ đã đối soát không được liệt kê trong đợt kê khai
- [ ] Given: Đợt kê khai A có 10 hồ sơ, trong đó 7 hồ sơ đã thuộc kỳ đối soát khác
- [ ] When: User tạo đối soát mới và đợt A hiển thị trong danh sách
- [ ] Then: Cột **Số lượng hồ sơ chưa đối soát** của đợt A hiển thị = 3 (chỉ tính hồ sơ chưa có trong bất kỳ kỳ đối soát nào)

---

### User Story: US-TL03 — Xem chi tiết

#### Scenario 3.5: BHYT không có tỷ lệ thù lao → hiển thị 0%
- [ ] Given: Kỳ đối soát chứa hồ sơ BHYT thuộc đối tượng không có tỷ lệ thù lao
- [ ] When: User xem chi tiết
- [ ] Then: Cột **Tỷ lệ thù lao (%)** của hồ sơ đó hiển thị **0%**
- [ ] And: Cột **Thù lao được hưởng** = 0

#### Scenario 3.6: Kỳ Đã đối soát → không hiển thị nút Chốt, Sửa, Xuất C17
- [ ] Given: Kỳ đối soát ở trạng thái `Đã đối soát`
- [ ] When: User truy cập chi tiết
- [ ] Then: Chỉ hiển thị nút **Quay lại** (không có Chốt, Sửa, Xuất C17)
- [ ] And: Giao diện ở chế độ Read-only hoàn toàn

#### Scenario 3.7: Phân trang bảng hồ sơ — dòng Tổng cộng tổng hợp tất cả trang
- [ ] Given: Kỳ đối soát có 50 hồ sơ (3 trang, 20 dòng/trang)
- [ ] When: User đang ở trang 1
- [ ] Then: Dòng Tổng cộng hiển thị tổng của **tất cả 50 hồ sơ**, không phải chỉ trang hiện tại

#### Scenario 3.8: Popup xác nhận chốt hiển thị đúng nội dung
- [ ] Given: User bấm Chốt đối soát
- [ ] Then: Popup hiển thị tiêu đề **"Chốt đối soát thù lao"** và nội dung **"Sau khi chốt đối soát, dữ liệu sẽ không thể chỉnh sửa. Bạn có chắc chắn muốn chốt kỳ đối soát này?"**

---

### User Story: US-TL04 — Sửa đối soát

#### Scenario 4.4: Hồ sơ lẻ chỉ hiển thị hồ sơ chưa thuộc kỳ nào
- [ ] Given: Popup Thêm hồ sơ lẻ đang mở
- [ ] Then: Danh sách chỉ gồm hồ sơ có trạng thái `Thành công` hoặc `CQ BHXH đang xử lý` **và** chưa thuộc bất kỳ kỳ đối soát nào **và** chưa nằm trong kỳ hiện tại

#### Scenario 4.5: Chọn nhiều hồ sơ lẻ cùng lúc
- [ ] Given: Popup Thêm hồ sơ lẻ
- [ ] When: User tích chọn tất cả → Bấm Xác nhận → Bấm Lưu
- [ ] Then: Tất cả hồ sơ được chọn thêm vào kỳ; dòng Tổng cộng được tính lại đầy đủ

#### Scenario 4.6: Bấm Quay lại từ màn hình Sửa → không lưu thay đổi
- [ ] Given: User đang ở màn hình Sửa và đã thêm/xóa hồ sơ nhưng chưa bấm Lưu
- [ ] When: User bấm **Quay lại**
- [ ] Then: Hệ thống quay về màn hình Xem chi tiết; dữ liệu không thay đổi

---

### User Story: US-TL05 — Xóa đối soát

#### Scenario 5.2: Bấm Hủy trong popup xóa → không xóa
- [ ] Given: Popup xác nhận xóa đang hiển thị
- [ ] When: User bấm **Hủy**
- [ ] Then: Popup đóng lại; kỳ đối soát vẫn tồn tại trong danh sách

---

## Section 4: Validation Rules

### Trường Tên đối soát (US-TL02)
- [ ] **Bắt buộc nhập**: Không được để trống (Ref: US-TL02)
- [ ] **Unique**: Tên đối soát phải là duy nhất trên toàn hệ thống. Nếu trùng, hiển thị inline: **"Tên đối soát đã tồn tại"** (Ref: US-TL02)

### Trường Tên đại lý thu (US-TL02)
- [ ] **Bắt buộc chọn**: Không được để trống (Ref: US-TL02)
- [ ] **Admin**: Dropdown hiển thị tất cả đại lý; **Đại lý thu**: tự điền và disable (Ref: US-TL02)

### Trường Cơ quan BHXH chi trả thù lao (US-TL02)
- [ ] **Bắt buộc chọn**: Không được để trống (Ref: US-TL02)
- [ ] **Enable khi**: Chỉ enable sau khi đã chọn Tên đại lý thu (Ref: US-TL02)
- [ ] **Giá trị mặc định**: CQ BHXH gắn với đại lý thu được chọn (Ref: US-TL02)

### Trường Giai đoạn đối soát (US-TL02)
- [ ] **Bắt buộc chọn**: Không được để trống (Ref: US-TL02)
- [ ] **Không chọn ngày tương lai**: Ngả chốt phải ≤ ngày hiện tại (Ref: US-TL02)
- [ ] **Định dạng**: dd/mm/yyyy - dd/mm/yyyy (Ref: US-TL02)

### Chọn đợt kê khai (US-TL02)
- [ ] **Tối thiểu 1 đợt**: Nếu bấm Tạo đối soát mà chưa chọn đợt nào, hiển thị: **"Vui lòng chọn tối thiểu 1 đợt kê khai."** (Ref: US-TL02)

### Bộ lọc giai đoạn đối soát (US-TL01)
- [ ] **Định dạng**: dd/mm/yyyy - dd/mm/yyyy (Ref: US-TL01)

---

## Section 5: Business Rules

### Tính tỷ lệ thù lao (US-TL02, US-TL03)
- [ ] **Công thức**: `Thù lao được hưởng = Số tiền đóng × Tỷ lệ thù lao (%)` (Ref: US-TL03)
- [ ] **Tra cứu tỷ lệ**: Lấy từ bảng `nv_chinh_sach_thu_lao` theo: `loai_nghiep_vu`, `loai_san_pham`, `phuong_thuc_dong`, `vung_dvhc` (từ tỉnh/TP nơi cư trú người tham gia), `phuong_an_dong`, `trang_thai = 1` (Ref: [Business Rule: Tỷ lệ thù lao])
- [ ] **BHYT không có tỷ lệ**: Nếu không tìm thấy bản ghi phù hợp trong danh mục, `Tỷ lệ thù lao = 0%` (Ref: US-TL03)
- [ ] **Vùng xác định**: Theo mã tỉnh/TP nơi cư trú người tham gia → `dm_donvi_hanhchinh.ma_vung` (I / II / III) (Ref: [Business Rule: Vùng DVHC])

### Điều kiện đợt kê khai hợp lệ (US-TL02)
- [ ] **Trạng thái**: Phải là `CQ BHXH đang xử lý` hoặc `Thành công` (Ref: US-TL02)
- [ ] **Còn hồ sơ chưa đối soát**: Số lượng hồ sơ chưa đối soát > 0 (Ref: US-TL02)
- [ ] **Trong giai đoạn**: Ngày gửi đợt nằm trong giai đoạn đối soát được chọn (Ref: US-TL02)
- [ ] **Đúng đại lý**: Đợt được nộp bởi đại lý thu đã chọn (Ref: US-TL02)
- [ ] **Tự động include tồn đọng**: Đợt kê khai cũ (trước giai đoạn) nhưng còn hồ sơ chưa đối soát tự động xuất hiện trong danh sách (Ref: US-TL02)

### Ràng buộc hồ sơ trong đối soát (US-TL02, US-TL04)
- [ ] **1 hồ sơ chỉ thuộc 1 kỳ**: Hồ sơ đã thuộc 1 kỳ đối soát bất kỳ (dù `Đang đối soát` hay `Đã đối soát`) không được thêm vào kỳ khác (Ref: [Business Rule: Exclusive Reconciliation])
- [ ] **Thêm hồ sơ lẻ (TL04)**: Hồ sơ phải có trạng thái `Thành công | CQ BHXH đang xử lý` + chưa thuộc kỳ nào + chưa trong kỳ hiện tại (Ref: US-TL04)

### Chốt đối soát (US-TL03)
- [ ] **Chỉ được chốt khi trạng thái = Đang đối soát** (Ref: US-TL03)
- [ ] **Sau khi chốt → Đã đối soát**: Dữ liệu không thể chỉnh sửa (Ref: US-TL03)

### Xóa đối soát (US-TL05)
- [ ] **Chỉ được xóa khi trạng thái = Đang đối soát**: Kỳ `Đã đối soát` không có tùy chọn Xóa (Ref: US-TL05)
- [ ] **Revert hồ sơ**: Sau khi xóa, các hồ sơ con được giải phóng (có thể thêm vào kỳ đối soát mới) (Ref: US-TL05)

### Sắp xếp dữ liệu
- [ ] **Danh sách đợt kê khai**: Sắp xếp từ cũ nhất đến mới nhất (Ref: US-TL02)
- [ ] **Danh sách biên lai trong C17**: Sắp xếp theo ngày biên lai từ cũ đến mới (Ref: US-TL03)

---

## Section 6: State Transitions

### Kỳ đối soát — State Machine

- [ ] `[Tạo mới]` → **DANG_DOI_SOAT** (Event: Tạo đối soát thành công) (Ref: US-TL02)
- [ ] **DANG_DOI_SOAT** → **DA_DOI_SOAT** (Event: Chốt đối soát) (Ref: US-TL03)
- [ ] **DANG_DOI_SOAT** → `[Deleted]` (Event: Xóa đối soát) (Ref: US-TL05)
- [ ] **DA_DOI_SOAT** → `[Không có transition tiếp]` (trạng thái cuối, immutable) (Ref: US-TL03)

### Hard Rules về trạng thái
- [ ] **Không thể sửa kỳ Đã đối soát**: Nút Sửa chỉ hiển thị khi `DANG_DOI_SOAT` (Ref: US-TL03, US-TL04)
- [ ] **Không thể xóa kỳ Đã đối soát**: Nút Xóa chỉ hiển thị khi `DANG_DOI_SOAT` (Ref: US-TL05)
- [ ] **Không thể xuất C17 kỳ Đã đối soát**: Nút Xuất C17 chỉ hiển thị khi `DANG_DOI_SOAT` (Ref: US-TL03)

---

## Section 7: Error Scenarios ⚙️ (OPTIONAL — Enhanced Coverage)

### Validation Errors (US-TL02)

#### Scenario: Tên đối soát trùng
- [ ] Given: Đã tồn tại kỳ đối soát tên "Đối soát tháng 1"
- [ ] When: User tạo kỳ mới với tên "Đối soát tháng 1"
- [ ] Then: Hiển thị lỗi inline: **"Tên đối soát đã tồn tại"** ngay tại field
- [ ] And: Kỳ đối soát không được tạo

#### Scenario: Chưa chọn đợt kê khai
- [ ] Given: User điền đủ thông tin chung nhưng không tích đợt kê khai nào
- [ ] When: Bấm **Tạo đối soát**
- [ ] Then: Hiển thị thông báo: **"Vui lòng chọn tối thiểu 1 đợt kê khai."**

#### Scenario: Giai đoạn đối soát chứa ngày tương lai
- [ ] Given: User chọn ngày kết thúc giai đoạn đối soát là ngày mai
- [ ] Then: Datepicker chặn không cho chọn (Ref: US-TL02)

### Business Errors (US-TL05)

#### Scenario: Xóa kỳ Đã đối soát (không khả thi)
- [ ] Given: Kỳ đối soát đang ở trạng thái `Đã đối soát`
- [ ] Then: Menu Thao tác không có tùy chọn **Xóa đối soát** (prevent at UI level)

---

## Section 8: Edge Cases 🎯 (OPTIONAL — Bonus Coverage)

### Boundary Conditions (US-TL02)
- [ ] **Đợt kê khai không có hồ sơ chưa đối soát (= 0)**: Không xuất hiện trong danh sách đợt kê khai (Ref: [Business Rule: Điều kiện đợt hợp lệ])
- [ ] **Kỳ đối soát chỉ có 1 hồ sơ**: Tổng cộng = giá trị của hồ sơ đó; xuất C17 chứa 1 dòng

### Concurrent Access
- [ ] **2 user cùng thêm hồ sơ vào 2 kỳ khác nhau**: UNIQUE constraint DB đảm bảo hồ sơ chỉ thuộc 1 kỳ, user thứ 2 nhận lỗi (Ref: [Business Rule: Exclusive Reconciliation])

### Data Integrity (US-TL05)
- [ ] **Xóa kỳ có nhiều hồ sơ**: Tất cả hồ sơ được revert; danh sách đợt kê khai liên quan cập nhật lại số lượng hồ sơ chưa đối soát (Ref: US-TL05)

---

## Section 9: Permissions & Access

### Portal APIs (Admin Đại lý thu)
- [ ] **Xem danh sách (TL01)**: Admin xem được tất cả kỳ đối soát của mọi đại lý; bộ lọc Đại lý thu hiển thị (Ref: US-TL01)
- [ ] **Tạo đối soát (TL02)**: Admin tạo cho bất kỳ đại lý nào (Ref: US-TL02)
- [ ] **Xem chi tiết (TL03)**: Admin xem bất kỳ kỳ nào (Ref: US-TL03)
- [ ] **Sửa đối soát (TL04)**: Admin sửa bất kỳ kỳ nào đang `DANG_DOI_SOAT` (Ref: US-TL04)
- [ ] **Chốt đối soát (TL03)**: Admin chốt bất kỳ kỳ nào đang `DANG_DOI_SOAT` (Ref: US-TL03)
- [ ] **Xóa đối soát (TL05)**: Admin xóa bất kỳ kỳ nào đang `DANG_DOI_SOAT` (Ref: US-TL05)

### Portal APIs (Đại lý thu)
- [ ] **Xem danh sách (TL01)**: Chỉ thấy kỳ do **chính mình tạo** (`created_by_user_id`); bộ lọc Đại lý thu **không** hiển thị (Ref: US-TL01)
- [ ] **Tạo đối soát (TL02)**: Được phép; trường Tên đại lý thu tự điền và disable (Ref: US-TL02)
- [ ] **Xem chi tiết (TL03)**: Chỉ kỳ do mình tạo (Ref: US-TL03)
- [ ] **Sửa (TL04)**: Chỉ kỳ do mình tạo + trạng thái `DANG_DOI_SOAT` (Ref: US-TL04)
- [ ] **Chốt (TL03)**: Chỉ kỳ do mình tạo + trạng thái `DANG_DOI_SOAT` (Ref: US-TL03)
- [ ] **Xóa (TL05)**: Chỉ kỳ do mình tạo + trạng thái `DANG_DOI_SOAT` (Ref: US-TL05)

### Data Scoping
- [ ] **Admin**: Xem toàn bộ hệ thống, không giới hạn đại lý (Ref: [Security Rule: Admin Scope])
- [ ] **Đại lý thu**: Chỉ thao tác kỳ có `created_by_user_id` = ID của mình (Ref: [Security Rule: Owner Scope])
- [ ] **API phải validate ownership**: Mọi thao tác trên kỳ đối soát phải kiểm tra quyền trước khi thực thi (Ref: [Hard Rule: Permissions])

---

## Section 10: External Capabilities

### Xuất C17-TS (US-TL03)
- [ ] **Capability**: Tạo file PDF theo mẫu C17-TS từ dữ liệu kỳ đối soát
- [ ] **Trigger**: User bấm nút **Xuất C17** khi kỳ ở trạng thái `DANG_DOI_SOAT`
- [ ] **Expected**: Tải xuống file `C17-TS.pdf` với đúng dữ liệu theo mẫu (CQ BHXH, tên đại lý, danh sách biên lai, tổng tiền bằng số và chữ)
- [ ] **Mapping mẫu**: (1) Tên CQ BHXH, (2) Để trống, (3) Tên đại lý thu, (4) STT tăng dần, (5) Ngày tạo kỳ đối soát, bảng biên lai, (6) Tổng số biên lai, (7) Tổng tiền, (8) Tổng tiền bằng chữ, (9) Tên đại diện đại lý (từ DB config), (10) Để trống

### Xuất DS chi tiết hồ sơ (US-TL03)
- [ ] **Capability**: Tạo file XLSX chứa toàn bộ danh sách hồ sơ trong kỳ đối soát
- [ ] **Trigger**: User click hyperlink **Tải file chi tiết hồ sơ**
- [ ] **Expected**: Tải xuống file `DS chi tiết hồ sơ.xlsx` đúng mẫu

### Danh mục CQ BHXH (US-TL02)
- [ ] **Capability**: Load danh sách cơ quan BHXH từ danh mục hệ thống
- [ ] **Trigger**: User mở popup Tạo đối soát sau khi chọn đại lý thu

---

## Section 11: Definition of Done ⭐

---

### User Story: US-TL01 — Xem danh sách đối soát thù lao

#### ✅ MANDATORY TIER 1: Happy Path Scenarios (100% Required)
- [ ] Scenarios 1.1, 1.2, 1.3, 1.4, 1.5 tất cả pass (Section 2)

#### ✅ MANDATORY TIER 2: Common Use Scenarios (100% Required)
- [ ] Scenarios 1.6, 1.7 pass (Section 3)

#### ✅ MANDATORY: Core Requirements (100% Required)
- [ ] Bộ lọc Đại lý thu chỉ hiển thị với Admin
- [ ] Phân trang mặc định 20 dòng/trang
- [ ] Menu thao tác khác nhau theo trạng thái
- [ ] Permissions validated (Admin: all, Đại lý thu: own only)

#### ⚙️ ENHANCED: Error Scenarios (Optional)
- [ ] Hiển thị "Không tìm thấy kết quả phù hợp." khi không có dữ liệu

---
**DoD Status**: [ ] Complete (✅ Tier 1 + Tier 2 + Core = 100%, ⚙️ 60%, 🎯 N/A)

---

### User Story: US-TL02 — Tạo đối soát thù lao

#### ✅ MANDATORY TIER 1: Happy Path Scenarios (100% Required)
- [ ] Scenarios 2.1, 2.2, 2.3, 2.4 tất cả pass (Section 2)

#### ✅ MANDATORY TIER 2: Common Use Scenarios (100% Required)
- [ ] Scenarios 2.5, 2.6, 2.7, 2.8 pass (Section 3)

#### ✅ MANDATORY: Core Requirements (100% Required)
- [ ] Tên đối soát UNIQUE — lỗi inline khi trùng
- [ ] Giai đoạn không cho chọn ngày tương lai
- [ ] Danh sách đợt reload khi thay đổi giai đoạn
- [ ] Validate tối thiểu 1 đợt kê khai được chọn
- [ ] Kỳ tạo xong → trạng thái `DANG_DOI_SOAT`
- [ ] Điều hướng sang màn hình chi tiết sau khi tạo
- [ ] Toast "Tạo đối soát thù lao thành công."

#### ⚙️ ENHANCED: Error Scenarios (Optional)
- [ ] Lỗi tên trùng inline "Tên đối soát đã tồn tại"
- [ ] Lỗi chưa chọn đợt "Vui lòng chọn tối thiểu 1 đợt kê khai."

---
**DoD Status**: [ ] Complete (✅ Tier 1 + Tier 2 + Core = 100%, ⚙️ 70%, 🎯 N/A)

---

### User Story: US-TL03 — Xem chi tiết đối soát

#### ✅ MANDATORY TIER 1: Happy Path Scenarios (100% Required)
- [ ] Scenarios 3.1, 3.2, 3.3, 3.4 tất cả pass (Section 2)

#### ✅ MANDATORY TIER 2: Common Use Scenarios (100% Required)
- [ ] Scenarios 3.5, 3.6, 3.7, 3.8 pass (Section 3)

#### ✅ MANDATORY: Core Requirements (100% Required)
- [ ] Công thức `Thù lao = Số tiền đóng × Tỷ lệ thù lao` đúng
- [ ] BHYT không có tỷ lệ → hiển thị 0%
- [ ] Dòng Tổng cộng = tổng tất cả trang (không phải trang hiện tại)
- [ ] Cột STT, Mã hồ sơ pinned cố định
- [ ] Sau chốt: nút Sửa/Chốt/Xuất C17 bị ẩn
- [ ] C17-TS export đúng mẫu với đúng dữ liệu
- [ ] XLSX export đúng danh sách hồ sơ

---
**DoD Status**: [ ] Complete (✅ Tier 1 + Tier 2 + Core = 100%, ⚙️ 50%, 🎯 N/A)

---

### User Story: US-TL04 — Sửa đối soát thù lao

#### ✅ MANDATORY TIER 1: Happy Path Scenarios (100% Required)
- [ ] Scenarios 4.1, 4.2, 4.3 tất cả pass (Section 2)

#### ✅ MANDATORY TIER 2: Common Use Scenarios (100% Required)
- [ ] Scenarios 4.4, 4.5, 4.6 pass (Section 3)

#### ✅ MANDATORY: Core Requirements (100% Required)
- [ ] Popup Thêm hồ sơ lẻ chỉ hiển thị hồ sơ hợp lệ (status + chưa đối soát + không trùng kỳ hiện tại)
- [ ] Tổng cộng được tính lại sau khi thêm/xóa hồ sơ
- [ ] Quay lại không lưu thay đổi pending
- [ ] Nút Thêm + cột Xóa chỉ hiển thị khi `DANG_DOI_SOAT`
- [ ] Permissions: chỉ người tạo (Đại lý thu) hoặc Admin được sửa

---
**DoD Status**: [ ] Complete (✅ Tier 1 + Tier 2 + Core = 100%, ⚙️ N/A, 🎯 N/A)

---

### User Story: US-TL05 — Xóa đối soát thù lao

#### ✅ MANDATORY TIER 1: Happy Path Scenarios (100% Required)
- [ ] Scenario 5.1 pass (Section 2)

#### ✅ MANDATORY TIER 2: Common Use Scenarios (100% Required)
- [ ] Scenario 5.2 pass (Section 3)

#### ✅ MANDATORY: Core Requirements (100% Required)
- [ ] Chỉ xóa được kỳ `DANG_DOI_SOAT`
- [ ] Kỳ `DA_DOI_SOAT` không có tùy chọn Xóa
- [ ] Sau xóa: hồ sơ con được revert (có thể đưa vào kỳ mới)
- [ ] Sau xóa: kỳ không còn trong danh sách
- [ ] Toast "Xóa đối soát thù lao thành công."
- [ ] Permissions: chỉ người tạo (Đại lý thu) hoặc Admin được xóa

#### 🎯 BONUS: Edge Cases (Optional)
- [ ] Xóa kỳ nhiều hồ sơ → tất cả revert đúng

---
**DoD Status**: [ ] Complete (✅ Tier 1 + Tier 2 + Core = 100%, ⚙️ 60%, 🎯 30%)

---

## Quality Gates

### Completeness ⭐
- [ ] Tất cả 5 User Stories có Happy Path Scenarios
- [ ] Tất cả 5 User Stories có Common Use Scenarios
- [ ] Sections 1-6, 9-11 được điền đầy đủ
- [ ] Tất cả validations từ URD có mặt trong Section 4
- [ ] Tất cả business rules từ URD có mặt trong Section 5
- [ ] State machine đầy đủ trong Section 6

### Traceability ⭐
- [ ] Mọi item có reference tag (Ref: US-TLxx)
- [ ] Tất cả URD content được trace vào checklist
- [ ] Không có requirement bị bỏ sót
