# [US-TL01] Xem danh sách đối soát thù lao

**As a** Admin Đại lý thu/ Đại lý thu  
**I want to** xem và quản lý danh sách các kỳ đối soát thù lao  
**So that** tôi có thể nắm bắt doanh thu, kiểm tra trạng thái đối soát thù lao với cơ quan BHXH của từng đại lý.

## 1. Chi tiết các trường dữ liệu và Validate
Note: Xem giao diện tại [Link](https://www.figma.com/design/vzviuk99H0k0SONYZmIUUY/VIVAS-BHXH?node-id=3587-3951&t=B8SlAmF0hGchrDNR-0)  

**A. Bộ lọc và Tìm kiếm**

* **Tìm kiếm**
  * Kiểu dữ liệu: Textbox.
  * Placeholder: "Tìm kiếm theo tên đối soát".

* **Giai đoạn đối soát**
  * Kiểu dữ liệu: Datepicker.
  * Định dạng: dd/mm/yyyy - dd/mm/yyyy.

* **Đại lý thu**: 
  * Hiển thị:  Chỉ hiển thị nếu role = Admin
  * Kiểu dữ liệu: Dropdown.
  * Giá trị: Danh sách (mã - tên) các Đại lý thu hộ trên hệ thống

* **Trạng thái**
  * Kiểu dữ liệu: Dropdown.
  * Giá trị: Đang đối soát | Đã đối soát.

**B. Bảng danh sách**

* Dữ liệu bảng bao gồm: STT, Tên đối soát, Giai đoạn đối soát, Tên đại lý thu, Cơ quan BHXH chi trả thù lao, Số lượng hồ sơ, Tổng tiền, Thù lao được hưởng, Trạng thái
  * Trong đó: **Số lượng hồ sơ** là tổng số lượng hồ sơ trong đợt đối soát đó
* Phân trang: Mặc định hiển thị 20 dòng/trang.

## 2. Trạng thái Control và Nút bấm

* **Nút "Tạo đối soát"**
  * Vị trí: Góc trên bên phải màn hình.
  * Hành vi: Click hiển thị popup Tạo đối soát thù lao. Chi tiết xem tại [US-TL02] Tạo đối soát thù lao.

* **Nút "Tìm kiếm"**
  * Hành vi: Thực hiện tìm kiếm dựa trên các tiêu chí tìm kiếm đã chọn.

* **Menu Thao tác**
  * Hành vi: Hiển thị menu phụ cho từng dòng bản ghi.
  * Menu phụ bao gồm các chức năng hiển thị theo trạng thái như sau: 
    * Ở trạng thái = Đang đối soát, gồm: Xem chi tiết, Sửa đối soát, Chốt đối soát, Xóa đối soát
    * Ở trạng thái = Đang đối soát, gồm: Xem chi tiết
## 3. Thông báo lỗi và Cảnh báo

* **Lỗi tìm kiếm:** "Không tìm thấy kết quả phù hợp.".