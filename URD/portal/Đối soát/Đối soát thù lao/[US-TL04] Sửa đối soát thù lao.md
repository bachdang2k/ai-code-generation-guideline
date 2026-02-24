# [US-TL04] Sửa đối soát thù lao

**As a** Admin Đại lý thu / Đại lý thu  
**I want to** chỉnh sửa danh sách hồ sơ trong kỳ đối soát (thêm/bớt hồ sơ vào kỳ đối soát)  
**So that** tôi có thể đảm bảo số liệu khớp với dữ liệu đối soát của CQ BHXH

## 1. Chi tiết các trường dữ liệu và Validate
Note: Xem giao diện tại [Link](https://www.figma.com/design/vzviuk99H0k0SONYZmIUUY/VIVAS-BHXH?node-id=3587-3951&t=B8SlAmF0hGchrDNR-0)  

Màn hình Sửa giữ nguyên cấu trúc hiển thị như màn hình Xem chi tiết, nhưng bổ sung: Button **Thêm hồ sơ lẻ**, cột thao tác **Xóa**

* **Nút "Thêm hồ sơ lẻ"**:
  * Chỉ hiển thị khi Trạng thái = **Đang đối soát**.
  * Hành vi: Mở Popup Thêm hồ sơ lẻ
  * Mô tả popup Thêm hồ sơ lẻ:
    * Tiêu đề: Thêm hồ sơ lẻ
    * Hiển thị bảng thông tin gồm: STT, Mã hồ sơ, Đợt kê khai, Người tham gia, Thủ tục, Trạng thái
    * Tìm kiếm: Kiểu dữ liệu: Searchbox. Cho phép tìm kiếm tương đối theo mã hồ sơ, người tham gia
    * Checkbox: Cho phép chọn lẻ hoặc chọn tất cả hồ sơ để đưa vào kỳ đối soát
    * Tổng số hồ sơ đã chọn: Hiển thị tổng số lượng hồ sơ đã được chọn phía dưới danh sách. Format: "Đã chọn: [number]"
    * Nút điều hướng: Hủy, Xác nhận

* **Cột Thao tác**:
  * Icon **Xóa**: Hiển thị ở cuối mỗi dòng.
  * Hành vi: Loại bỏ hồ sơ khỏi kỳ đối soát hiện tại
  * Pinned cố định cột Thao tác

## 2. Trạng thái Control và Nút bấm

* **Nút "Quay lại"**
  * Hành vi: Hủy bỏ các thao tác đang sửa, quay về màn hình Xem chi tiết.

* **Nút "Lưu"**
  * Hành vi: Cập nhật dữ liệu vào cơ sở dữ liệu.

## 3. Cách hệ thống xử lý

* **Logic "Thêm hồ sơ lẻ"**:
  * Khi bấm nút, hệ thống hiển thị danh sách các hồ sơ thỏa mãn:
    1. Trạng thái = Thành công | CQ BHXH đang xử lý.
    2. Chưa thuộc bất kỳ kỳ đối soát nào (cả trạng thái =  Đang đối soát | Đã đối soát).
    3. Chưa nằm trong danh sách hiện tại của kỳ này.
  * Sau khi thêm, hệ thống tự động tính lại dòng Tổng cộng số tiền đóng, Thù lao được hưởng

## 4. Thông báo

* **Xóa hồ sơ khỏi kỳ đối soát**: "Đã loại bỏ hồ sơ khỏi kỳ đối soát."
* **Cập nhật kỳ đối soát thành công**: "Cập nhật đối soát thù lao thành công."