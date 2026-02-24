# [US-TL02] Tạo đối soát thù lao

**As a** Admin Đại lý thu/ Đại lý thu  
**I want to** tạo kỳ đối soát thù lao dựa trên các đợt kê khai đã hoàn thành  
**So that** hệ thống có thể tổng hợp hồ sơ trong các đợt kê khai được chọn để tính thù lao và đối chiếu với cơ quan BHXH.

## 1. Chi tiết các trường dữ liệu và Validate
Note: Xem giao diện tại [Link](https://www.figma.com/design/vzviuk99H0k0SONYZmIUUY/VIVAS-BHXH?node-id=3587-3951&t=B8SlAmF0hGchrDNR-0)  

### A. Thông tin chung

* **Tên đối soát**
  * Kiểu dữ liệu: Textbox. Tên đối soát là unique. Nếu trùng hiển thị inline: "Tên đối soát đã tồn tại
  * Ràng buộc: Bắt buộc nhập.
  * Hintext: Nhập tên đối soát

* **Tên đại lý thu**
  * Kiểu dữ liệu: Dropdown.
  * Logic:
    * Nếu role = Admin: Hiển thị danh sách tên các Đại lý thu hộ trên hệ thống
    * Nếu role = Đại lý thu: Hiển thị là tên đại lý đó và disable
  * Ràng buộc: Bắt buộc chọn

* **Cơ quan BHXH chi trả thù lao**
  * Enable khi đã chọn Tên đại lý thu
  * Kiểu dữ liệu: Dropdown. Giá trị lấy từ danh mục cơ quan BHXH
  * Ràng buộc: Bắt buộc chọn
  * Giá trị mặc định: Hiển thị tên CQ BHXH gắn với đại lý thu 

* **Giai đoạn đối soát**
  * Kiểu dữ liệu: Datepicker.
  * Định dạng: dd/mm/yyyy - dd/mm/yyyy
  * Ràng buộc: Bắt buộc chọn.
  * Validate: Chặn không cho chọn ngày tương lai
  * Tác động: Khi thay đổi ngày, danh sách đợt kê khai bên dưới sẽ được tải lại (Reload).

### B. Danh sách đợt kê khai

* **Thanh tìm kiếm**: Searchbox. Cho phép tìm tìm kiếm tương đối theo đợt kê khai

* **Bảng danh sách**
  * Hiển thị bảng thông tin gồm: STT, Đợt kê khai, Thủ tục, Số lượng hồ sơ chưa đối soát
  * Dữ liệu: Hệ thống chỉ hiển thị các đợt kê khai thỏa mãn đồng thời các điều kiện sau:
    * 1 - Trạng thái = **CQ BHXH đang xử lý, Thành công**
    * 2 - Số lượng hồ sơ chưa đối soát >0
    * 3 - Ngày gửi của đợt kê khai nằm trong giai đoạn đối soát được chọn ở mục A
    * 4 - Được nộp bởi Đại lý thu được chọn ở mục A
    * Lưu ý: Hệ thống tự động lấy thêm vào danh sách các đợt kê khai có trạng thái **CQ BHXH đang xử lý, Thành công** nằm ở khoảng thời gian **trước** giai đoạn đối soát nhưng vẫn còn tồn đọng hồ sơ chưa được đối soát (Số lượng hồ sơ chưa đối soát > 0).
  * Dữ liệu đợt kê khai sắp xếp theo đợt kê khai từ cũ nhất đến mới nhất

* **Checkbox**: 
  * Cho phép chọn lẻ hoặc chọn tất cả hồ sơ để đưa vào đợt đối soát.
  * Tối thiểu tích chọn 1 đợt kê khai
  * Nếu người dùng bấm**Tạo đối soá** mà chưa chọn đợt nào, hiển thị thông báo: "Vui lòng chọn tối thiểu 1 đợt kê khai."


## 2. Trạng thái Control và Nút bấm

* **Nút "Hủy"**
  * Hành vi: Đóng popup, không lưu dữ liệu.

* **Nút "Tạo đối soát"**
  * Hành vi: 
    1. Tạo và lưu thông tin kỳ đối soát.
    2. Điều hướng người dùng sang màn hình **Xem chi tiết đối soát**

## 3. Cách hệ thống xử lý

* **Logic khởi tạo đối soát**:
  * Hệ thống gen ra danh sách hồ sơ thuộc các đợt kê khai đã được chọn 
  * Kỳ đối soát sau khi tạo thành công sẽ có trạng thái là **"Đang đối soát"**.

* **Lưu ý**:
  * Một hồ sơ chỉ được nằm trong duy nhất 1 giai đoạn đối soát. Nếu hồ sơ đã nằm trong 1 giai đoạn đối soát (cả trạng thái là Đang đối soát và Đã đối soát), không được liệt kê vào các giai đoạn đối soát khác.

## 4. Thông báo

* **Thông báo toast thành công**: "Tạo đối soát thù lao thành công."