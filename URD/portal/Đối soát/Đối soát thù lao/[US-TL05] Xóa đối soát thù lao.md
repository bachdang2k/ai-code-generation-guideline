# [US-TL05] Xóa đối soát thù lao

**As a** Admin Đại lý thu / Đại lý thu  
**I want to** xóa bỏ các kỳ đối soát được lập sai hoặc chưa chốt số liệu  
**So that** tôi có thể xóa kỳ đối soát đó để tạo lại kỳ đối soát mới chính xác hơn.

## 1. Các điểm kích hoạt 
Note: Xem giao diện tại [Link](https://www.figma.com/design/vzviuk99H0k0SONYZmIUUY/VIVAS-BHXH?node-id=3587-3951&t=B8SlAmF0hGchrDNR-0)  

**Nút "Xóa đối soát"**:  
  * Vị trí: Menu Thao tác tại màn Danh sách đối soát hàng ngày
  * Chỉ hiển thị nếu Trạng thái = **Đang đối soát**
  * Click hiển thị popup xác nhận xóa đối soát: 
    * **Tiêu đề**: Xóa đối soát thù lao
    * **Nội dung**: "Bạn có chắc chắn muốn xóa đối soát này?"
    * **Nút điều hướng**: "Hủy" và "Đồng ý".

## 2. Cách hệ thống xử lý

* Hệ thống xóa kỳ đối soát khỏi hệ thống, bảng danh sách không còn hiển thị kỳ đối soát vừa xóa
* Các hồ sơ con thuộc kỳ đối soát đó được revert về trạng thái trước đó 

## 3. Thông báo và Cảnh báo

* **Xóa đối soát thành công**: Hiển thị toast: *"Xóa đối soát thù lao thành công."*.