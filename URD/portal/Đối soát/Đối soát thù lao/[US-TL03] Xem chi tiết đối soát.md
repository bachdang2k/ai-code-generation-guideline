# [US-TL03] Xem chi tiết đối soát

**As a** Admin Đại lý thu / Đại lý thu  
**I want to** xem chi tiết bảng kê tính toán thù lao của một kỳ đối soát  
**So that** tôi có thể kiểm tra lại tính chính xác của từng khoản thù lao theo từng hồ sơ trước khi thực hiện chốt đối soát.

## 1. Chi tiết các trường dữ liệu
Note: Xem giao diện tại [Link](https://www.figma.com/design/vzviuk99H0k0SONYZmIUUY/VIVAS-BHXH?node-id=3587-3951&t=B8SlAmF0hGchrDNR-0)  

Giao diện hiển thị ở chế độ **Read-only**, chia làm 2 phần chính: Thông tin đối soát và Danh sách hồ sơ.

### A. Thông tin đối soát

Hiển thị các trường thông tin: Tên đối soát, Giai đoạn đối soát, Tên đại lý thu, Cơ quan BHXH chi trả thù lao.

### B. Danh sách hồ sơ

Bảng liệt kê chi tiết các hồ sơ thuộc kỳ đối soát:

* **Các cột dữ liệu gồm**: STT, Mã hồ sơ, Người tham gia, Thủ tục, Tỉnh/TP, Vùng, Phương án đóng, Số biên lai, Số tiền đóng, Tỷ lệ thù lao (%), Thù lao được hưởng, Đại lý thu hộ, Nhân viên thu
  * Bảng danh sách pinned cố định cột STT, Mã hồ sơ
  * Phân trang: Mặc định hiển thị 20 dòng/trang.
  * Vùng và Tỷ lệ thù lao lấy theo file đính kèm: [Link](https://docs.google.com/spreadsheets/d/1PgfnOjgm_TKKZ0uw87K-8slEzsgnOAVR/edit?usp=sharing&ouid=101886527944435650480&rtpof=true&sd=true)
  * Với các đối tượng của BHYT không có tỷ lệ thù lao, để mặc định = 0%


* **Dòng Tổng cộng**:
  * Hiển thị tổng của cột **Số tiền đóng** của tất cả các kết quả ở các trang
  * Hiển thị tổng của cột **Thù lao được hưởng** của tất cả các kết quả ở các trang
  * Định dạng: Chữ in đậm, màu đỏ để làm nổi bật.

* **Tải file chi tiết hồ sơ**:
  * Kiểu dữ liệu: Hyper link
  * Hành vi: Click tải xuống file chi tiết hồ sơ trong kỳ đối soát đó
    * Tên file: DS chi tiết hồ sơ
    * Định dạng: .xlsx
    * Mẫu file: [Link](https://docs.google.com/spreadsheets/d/1id19VEbT4pYJ6d9794-zjS8n4M0W-ve6/edit?usp=sharing&ouid=101886527944435650480&rtpof=true&sd=true)

## 2. Trạng thái Control và Nút bấm

* **Nút "Quay lại"**
  * Hành vi: Quay về màn hình Danh sách đối soát thù lao.

* **Nút "Chốt đối soát"**
  * Chỉ hiển thị khi Trạng thái = **Đang đối soát**.
  * Hành vi: Click hiển thị popup Chốt đối soát. Sau khi chốt, trạng thái chuyển sang **Đã đối soát**.
  * Popup Chốt đối soát:
    * Tiêu đề: Chốt đối soát thù lao
    * Nội dung: "Sau khi chốt đối soát, dữ liệu sẽ không thể chỉnh sửa. Bạn có chắc chắn muốn chốt kỳ đối soát này? "

* **Nút "Xuất C17"**
  * Chỉ hiển thị khi Trạng thái = **Đang đối soát**.
  * Hành vi: Tải xuống C17-TS chứa thông tin của hồ sơ thuộc kỳ đối soát đó
    * Định dạng: .pdf
    * Tên file: C17-TS
    * Mẫu file: [Link](https://docs.google.com/document/d/1D12N_ecN01yjYU0gKiNEpakH5eFgEezK/edit?usp=sharing&ouid=101886527944435650480&rtpof=true&sd=true)
* Mô tả điền mẫu C17-TS:  
  * (1): Tên CQ BHXH chi trả thù lao
  * (2): Để trống
  * (3): Tên đại lý thu
  * (4): Là số thứ tự tăng dần, bắt đầu từ 01
  * (5): Là ngày tạo kỳ đối soát thù lao
  * Bảng thông tin: sắp xếp theo ngày biên lai từ cũ nhất đến mới nhất
    * STT: Tăng dần bắt đầu từ 01
    * Quyển số: Để trống
    * Số: Là số biên lai
    * Ngày biên lai: Là ngày trên biên lai
    * BHXH: Điền số tiền trên biên lai. Chỉ điền nếu biên lai là của BHXH
    * BHYT: Điền số tiền trên biên lai. Chỉ điền nếu biên lai là của BHYT
    * Tổng số: Bằng số tiền (BHXH + BHYT) của dòng tương ứng
  * (6): Tổng số lượng biên lai
  * (7): Tổng số tiền của tất cả các biên lai trong kỳ đối soát
  * (8): Tổng số tiền viết bằng chữ
  * (9): Tên người đại diện của Đại lý thu (lấy dữ liệu được cấu hình từ DB)
  * (10): Để trống

* **Nút "Sửa"**
  * Điều kiện hiển thị: Chỉ hiển thị khi Trạng thái = **Đang đối soát**.
  * Hành vi: Mở chức năng Sửa đối soát thù lao. Chi tiết xem [US-TL04] - Sửa đối soát thù lao

## 3. Cách hệ thống xử lý

* **Logic tính toán**:
   `Thù lao được hưởng` = `Số tiền đóng` * `Tỷ lệ thù lao (%)`.

## 4. Thông báo

* **Thông báo toast thành công khi chốt đối soát**: "Chốt đối soát thù lao thành công."