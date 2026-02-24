# PROMPT PHÁT TRIỂN HỆ THỐNG QUẢN LÝ THU HỘ BHXH TỰ NGUYỆN VÀ BHYT HỘ GIA ĐÌNH

## 1. TỔNG QUAN HỆ THỐNG

Phát triển hệ thống quản lý thu hộ BHXH tự nguyện và BHYT hộ gia đình với 2 thành phần chính:
- **Web Admin**: Dành cho Đại lý thu hộ và nhân viên thu hộ (giao diện web responsive)
- **Java Backend**: Nghiệp vụ hệ thống (REST API cho web admin)

Sử dụng REST API cho giao tiếp giữa frontend và backend. Ưu tiên phát triển Java Backend trước, sau đó mới phát triển Web Admin. 
Backend sẽ cần kết nối với Bảo hiểm xã hội Việt Nam thông qua API của đối tác IVAN để xử lý nghiệp vụ. Kết nối này sẽ được phát triển sau, nhưng cần chuẩn bị sẵn các class cần thiết trước.

# Context & Constraints

- **Tech Stack:** Java Spring Boot, MySQL_DB, REST API.
- **Ngôn ngữ code:** Tên biến, Class, Database, Comment khuyến nghị ưu tiên dùng **Tiếng Việt** trừ những trường đặc biệt như ID.
- **Tiền tệ:** Tính toán chính xác đến hàng đơn vị (VNĐ), sử dụng `BigDecimal`.


## 2. QUY TẮC VỀ QUẢN LÝ DATABASE (BẮT BUỘC):
**KHÔNG được tác động đến cấu trúc database:**
- ❌ **KHÔNG** được bổ sung các script DDL (CREATE TABLE, ALTER TABLE, DROP TABLE, ADD COLUMN, MODIFY COLUMN, v.v.) vào các file migration (`src/main/resources/db/migration/V*.sql`)
- ❌ **KHÔNG** được tạo file migration mới
- ❌ **KHÔNG** được thay đổi Entity JPA làm ảnh hưởng đến schema database (thêm/xóa/sửa cột, thêm/xóa bảng)

**ĐƯỢC PHÉP làm:**
- ✅ **Gợi ý thiết kế database**: Có thể đề xuất cấu trúc bảng, cột, quan hệ cho người dùng tự triển khai
- ✅ **Tạo data mock/seed data**: Có thể tạo các script INSERT để thêm dữ liệu mẫu
- ✅ **Viết code Java**: Entity, Repository, Service, Controller - miễn là KHÔNG thay đổi cấu trúc DB hiện tại
- ✅ **Sử dụng cấu trúc DB hiện có**: Làm việc với các bảng/cột đã tồn tại

**Lý do:** Database được thiết kế và quản lý bên ngoài bởi team riêng.

## 4. YÊU CẦU JAVA BACKEND
Tên thư mục dự án:  bhxh-backend
Include: Java 21, Spring Boot 3.x, JPA, MySQL driver, Flyway, springdoc-openapi, Lombok, Spring Boot Test.
Authentication and authorization use JWT token provided by other microservice, so only decoding is needed. Use Spring Security to authorize API access for each endpoint. For first phase, let user access all endpoints.
Base package:  vn.vivas.bh
Generate schema.sql and data.sql for database initialization with Flyway.
Config:
- application.yml: server.port={{port}}, MySQL datasource, Hibernate MySQL dialect, ddl-auto=validate, Flyway enabled, Swagger paths
- logback.xml: basic console/file logging and auto-rotate log.
Cấu trúc dự án backend:  Ưu tiên sự đơn giản, dễ đọc, dễ hiểu, không tạo nhiều Java Interface trừ khi thực sự cần thiết.
- Mở cấu hình CORS để cho phép truy cập từ mọi origin
- các nghiệp vụ liên quan đến upload file phía Client upload file trước rồi gửi url cho phía BE

## 4.0. API PAGINATION

- Tất cả các nghiệp vụ phân trang thực hiện mới, các req cần đồng nhất sử dụng: vn.vivas.bh.request.PaginationRequest, response: vn.vivas.bh.response.PageResponse

## 4.1. EXCEPTION HANDLING

### MessageResponseDict Enum
- **BẮT BUỘC**: Tất cả response codes phải được định nghĩa trong enum `MessageResponseDict`
- **Quy tắc mã code**:
  - `code = 0`: Thành công
  - `code = 100000-400000`: Các mã lỗi nghiệp vụ
- **HttpStatus**: Mặc định là `HttpStatus.OK` trừ khi có yêu cầu đặc biệt (ví dụ: NOT_FOUND, UNAUTHORIZED)
- **Format khai báo enum**:
  ```java
  // Success with code 0
  ADD_SUCCESS(HttpStatus.OK, 0, "Thêm mới thành công"),
  UPDATE_SUCCESS(HttpStatus.OK, 0, "Cập nhật thành công"),
  
  // Error with code in range 100000-400000
  BUSINESS_NOT_FOUND(HttpStatus.NOT_FOUND, 100000, "Doanh nghiệp không tồn tại!"),
  INVALID_INPUT(HttpStatus.OK, 100001, "Dữ liệu đầu vào không hợp lệ"),
  
  // With dynamic parameters using %s
  OBJECT_NOT_FOUND(HttpStatus.NOT_FOUND, 100002, "%s bạn chọn không tồn tại")
  ```
- **KHÔNG BAO GIỜ** hardcode response code/message trực tiếp trong code, luôn sử dụng enum

### Custom Exceptions
- Tạo custom exception extends `RuntimeException` với các thuộc tính:
  - `MessageResponseDict responseDict`: chứa code, message, HTTP status
  - `List<String> params`: tham số động cho message template
- Các loại exception: `BusinessException`, `NotRollbackException`, `AppException`, `ApiCallException`
- Throw exception ngay khi validate fail, không return error codes
- **BẮT BUỘC** sử dụng `MessageResponseDict` enum khi throw exception:
  ```java
  // Good
  throw new BusinessException(MessageResponseDict.BUSINESS_NOT_FOUND);
  throw new BusinessException(MessageResponseDict.OBJECT_NOT_FOUND, List.of("Doanh nghiệp"));
  
  // Bad - KHÔNG hardcode message
  throw new BusinessException("Business not found");
  ```

### Exception Controller (@ControllerAdvice)
- Tạo `ExceptionController` với `@ControllerAdvice` để bắt tất cả exceptions
- Mỗi exception type có 1 `@ExceptionHandler` riêng
- Return `ResponseEntity<ResponseCommon<?>>` với structure:
  ```java
  ResponseCommon {
    code: Integer,
    message: String,
    data: T
  }
  ```
- Với `AuthenticationException` và `AccessDeniedException`: re-throw để SecurityFilterChain xử lý
- Log error với `log.error()` cho system errors, `log.warn()` cho business errors

### Response Structure
- Success: HTTP 200 với `code=0`, `message` từ `MessageResponseDict`
- Business Error: HTTP status phù hợp (400, 404, 409...) hoặc mặc định OK với `code` (100000-400000) và `message` từ exception
- System Error: HTTP 500 với generic error message

### Checklist: MessageResponseDict Usage
✅ **BẮT BUỘC thực hiện**:
- [ ] Tất cả response codes được định nghĩa trong `MessageResponseDict` enum
- [ ] Success code = 0
- [ ] Error code trong range 100000-400000
- [ ] HttpStatus mặc định là `HttpStatus.OK` trừ khi cần đặc biệt (NOT_FOUND, UNAUTHORIZED, etc.)
- [ ] Throw exception với `MessageResponseDict` enum, không hardcode message
- [ ] Sử dụng `List<String> params` cho dynamic message với `%s`

❌ **KHÔNG BAO GIỜ làm**:
- Hardcode response code/message trực tiếp: ~~`throw new Exception("Error")`~~
- Return error code thay vì throw exception: ~~`return -1;`~~
- Tạo ResponseCommon manually: ~~`new ResponseCommon(400, "Error")`~~

## 4.2. LOGGING STANDARDS

### Log Format
```
[{userId}/{businessId}] {action} - {details}
```
Ví dụ: `[123/456] Update brandname status - brandnameId=789, status=ACTIVE`

### Log Levels
- `log.error()`: System errors, exceptions với stacktrace
  ```java
  log.error("Exception handleRuntimeException: {}", ex.getMessage(), ex);
  ```
- `log.warn()`: Business errors, validation failures (không có stacktrace)
  ```java
  log.warn("BusinessException: http-status={} code={} message={}", status, code, message);
  ```
- `log.info()`: Business operations, state changes
  ```java
  log.info("Updated {} dependent brandnames", toUpdate.size());
  log.info("Completed updateSystemBrandName for labelId={}", labelId);
  ```
- `log.debug()`: Detailed debug information (tắt ở production)

### Logging Rules
- Log START và END của các transaction quan trọng
- Log tất cả state changes (status updates, data modifications)
- Không log sensitive data (passwords, tokens, personal info)
- Log parameters quan trọng để trace được flow
- Với loop xử lý nhiều items: log summary count thay vì từng item
- Log failed operations với đầy đủ context để debug

### Best Practices
```java
// Good: Context đầy đủ
log.info("[{}] Approved brandname - brandnameId={}, expiredAt={}", 
    userId, labelId, expiredAt);

// Good: Summary thay vì chi tiết từng item
log.info("Updated {} business labels with expiration reason", toUpdate.size());

// Bad: Thiếu context
log.info("Updated successfully");

// Bad: Log sensitive data
log.info("User password: {}", password);
```
## 4.3. SERVICE LAYER PATTERNS

### Transaction Management
- Sử dụng `@Transactional` cho methods có multiple DB operations
- Hoặc dùng `TransactionTemplate` cho programmatic control:
  ```java
  transactionTemplate.executeWithoutResult(status -> {
      // DB operations
  });
  ```
- Business validation trước khi vào transaction block

### Method Structure
```java
public Response methodName(Request request) {
    // 1. Validate authentication/authorization
    AccountInfo user = AuthenticationService.getCurrentUserRequired();
    
    // 2. Validate input & business rules
    validateInput(request);
    
    // 3. Load entities
    Entity entity = repository.findById(id)
        .orElseThrow(() -> new BusinessException(MessageResponseDict.NOT_FOUND));
    
    // 4. Business logic & state changes
    entity.setState(newState);
    
    // 5. Persist changes (trong transaction nếu cần)
    repository.save(entity);
    
    // 6. Async operations (notifications, emails...)
    CompletableFuture.runAsync(() -> sendEmail());
    
    // 7. Return response
    return mappingHelper.map(entity, Response.class);
}
```

### Validation Pattern
- Tạo private methods `validateXxx()` để reuse logic
- **BẮT BUỘC** throw `BusinessException` với `MessageResponseDict` enum phù hợp
- Validate theo thứ tự: required fields → format → business rules
- **KHÔNG BAO GIỜ** hardcode error message, luôn định nghĩa trong `MessageResponseDict` trước
```java
private void validateInput(Request request) {
    if (StringUtils.isEmpty(request.getName())) {
        // Good - Sử dụng enum đã được định nghĩa
        throw new BusinessException(MessageResponseDict.FIELD_REQUIRED);
    }
    if (request.getName().matches(INVALID_PATTERN)) {
        // Good - Sử dụng enum với params động
        throw new BusinessException(MessageResponseDict.INVALID_FORMAT, List.of("Tên"));
    }
    
    // Bad - KHÔNG làm thế này
    // throw new BusinessException("Invalid name");
    // return ResponseEntity.badRequest().body("Error");
}
```
- **Quy trình thêm validation mới**:
  1. Định nghĩa enum trong `MessageResponseDict` với code từ 100000-400000
  2. Sử dụng enum đó để throw exception trong validation
  ```java
  // Step 1: Thêm vào MessageResponseDict.java
  INVALID_PHONE_NUMBER(HttpStatus.OK, 100050, "Số điện thoại không hợp lệ"),
  
  // Step 2: Sử dụng trong validation
  if (!isValidPhone(request.getPhone())) {
      throw new BusinessException(MessageResponseDict.INVALID_PHONE_NUMBER);
  }
  ```

## 4.4. API RESPONSE CONVENTIONS

### Response Code Standards
- **BẮT BUỘC**: Tất cả API responses phải sử dụng `MessageResponseDict` enum
- **Success response**: `code = 0`, HTTP status mặc định `OK (200)`
- **Error response**: `code = 100000-400000`, HTTP status mặc định `OK (200)` trừ khi có yêu cầu đặc biệt
- **Ví dụ khai báo trong MessageResponseDict**:
  ```java
  // Success - code=0, httpStatus=OK
  ADD_SUCCESS(HttpStatus.OK, 0, "Thêm mới thành công"),
  DELETE_SUCCESS(HttpStatus.OK, 0, "Xóa thành công"),
  SUCCESS(HttpStatus.OK, 0, "Thành công"),
  
  // Error - code=100000-400000
  BUSINESS_NOT_FOUND(HttpStatus.NOT_FOUND, 100000, "Doanh nghiệp không tồn tại!"),
  CANNOT_DELETE_CONSTRAINT(HttpStatus.OK, 100001, "Không thể xóa %s vì ràng buộc dữ liệu"),
  OBJECT_NOT_FOUND(HttpStatus.NOT_FOUND, 100002, "%s bạn chọn không tồn tại"),
  ```

### Success Response
- HTTP 200 với `ResponseCommon` chứa data
- **BẮT BUỘC** sử dụng `MessageResponseDict` enum:
  ```java
  // Good
  return ResponseUtils.create(MessageResponseDict.ADD_SUCCESS, data);
  return ResponseUtils.ok(MessageResponseDict.SUCCESS, data);
  
  // Bad - KHÔNG hardcode
  return ResponseEntity.ok(new ResponseCommon(0, "Success", data));
  ```

### Error Response
- HTTP status code phù hợp (400, 404, 409, 500...) hoặc mặc định OK nếu không có yêu cầu đặc biệt
- `ResponseCommon` với `code` và `message` từ `MessageResponseDict` enum
- Không expose internal error details ra client
- **BẮT BUỘC** throw exception với `MessageResponseDict`:
  ```java
  // Good
  throw new BusinessException(MessageResponseDict.BUSINESS_NOT_FOUND);
  
  // Bad - KHÔNG hardcode
  throw new RuntimeException("Not found");
  ```

### Pagination Response
```java
PagingResponse {
    paging: { pageIndex, pageSize, totalPages, totalElements },
    data: List<T>,
    extraInfo: Object
}
```

## 4.5. CODING CONVENTIONS

### Naming
- Service methods: verb + noun (`createBrandName`, `updateStatus`, `getBrandNameList`)
- Private helper methods: `validateXxx`, `buildXxx`, `sendXxx`, `processXxx`
- Boolean fields: `isXxx`, `hasXxx` (không dùng `getXxx` cho boolean)

### Code Organization
- Group related private methods gần method sử dụng chúng
- Constants ở đầu class
- Dependencies injection qua constructor
- Tránh methods quá dài (>100 lines), extract sub-methods

### Comments
- Không comment những gì code đã nói rõ
- Comment cho business logic phức tạp
- Comment cho workarounds hoặc temporary solutions
- TODO comments phải có JIRA ticket hoặc context rõ ràng

## 4.6. SECURITY & AUTHORIZATION

### Authentication
- Lấy current user: `AuthenticationService.getCurrentUserRequired()`
- Throw exception nếu user null hoặc không có quyền
- Validate ownership trước khi cho phép modify data

### Data Access Control
- Filter data theo businessId/agentId của user
- Admin có thể access all data
- Agent chỉ access data của businesses thuộc agent
- Client chỉ access data của business riêng


## 5. YÊU CẦU FRONTEND
- Tên thư mục:  bhxh-fe-demo
- Sử dụng công nghệ tùy ý, miễn là đảm bảo responsive.
- Giao diện Frontend được sử dụng với mục đích demo các tính năng của hệ thống chứ không phải là giao diện chính thức. Do vậy, không cần thiết phải tạo giao diện phức tạp mà ưu tiên chức năng hoạt động được các form đầy đủ thông tin.