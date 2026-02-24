## Thiết kế Cơ sở dữ liệu (Chi tiết bắt buộc)
Yêu cầu: Thiết kế các Entity JPA (và bảng MariaDB tương ứng) chính xác theo cấu trúc dưới đây. Sử dụng `Snake Case` cho tên cột trong DB, `Camel Case` cho biến Java.
### Quy tắc chung:
- Tên bảng/cột: ưu tiên Tiếng Việt không dấu (snake_case) trừ các cột đặc biệt như ID, timestamp
- Tiền tố nhóm: `dm_` (danh mục), `ht_` (hệ thống), `nv_` (nghiệp vụ), `ls_` (lịch sử)
- Sử dụng `mo_rong` (JSON) cho các trường linh hoạt
- Khóa chính các bảng danh mục là `ma` (VARCHAR) để đồng bộ với dữ liệu BHXH

### ⚠️ QUY TẮC VỀ QUẢN LÝ DATABASE (BẮT BUỘC):
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

### Cấu trúc CSDL chi tiết:

```sql
create table agency
(
    id                   int auto_increment
        primary key,
    name                 varchar(255)   null,
    ascii_name           varchar(255)   null,
    description          varchar(255)   null,
    email                varchar(255)   null,
    phone                varchar(20)    null,
    region_id            int            null,
    city_id              int            null,
    district_id          int            null,
    ward_id              int            null,
    address              varchar(255)   null,
    status               int default 10 null,
    parent_id            int            null,
    admin_id             int            null,
    level                int default 1  null,
    path                 varchar(255)   null,
    created_at           int            null,
    updated_at           int            null,
    created_by           int            null,
    updated_by           int            null,
    agency_code          varchar(255)   null comment 'Mã đại lý',
    agent_level_id       int            null comment 'Cấp đại lý',
    social_insurance     varchar(255)   null comment 'Cơ quan BHXH',
    bank_account         varchar(255)   null comment 'Số tài khoản',
    tax_code             varchar(255)   null comment 'Mã số thuế',
    username             varchar(255)   null comment 'Tên tài khoản',
    fullname             varchar(255)   null,
    representative_name  varchar(255)   null comment 'Tên đại diện',
    representative_phone varchar(255)   null comment 'Số điện thoại đại diện',
    representative_email varchar(255)   null comment 'Email đại diện',
    bank_name            varchar(255)   null comment 'Tên ngân hàng',
    bank_owner           varchar(255)   null comment 'Chủ tài khoản',
    bank_city_id         int            null comment 'Tỉnh/Thành phố mở tài khoản'
);

create table dm_benh
(
    id          bigint auto_increment comment 'Thứ tự'
        primary key,
    ma_benh     varchar(20)                        not null comment 'Mã bệnh nội bộ',
    ten_benh    varchar(255)                       not null comment 'Tên bệnh',
    ma_icd      varchar(20)                        not null comment 'Mã ICD',
    ghi_chu     varchar(255)                       null comment 'Ghi chú',
    last_update datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP,
    constraint uk_dm_benh_ma_benh
        unique (ma_benh)
)
    comment 'Danh mục bệnh' collate = utf8mb4_unicode_ci;

create index idx_dm_benh_ma_icd
    on dm_benh (ma_icd);

create index idx_dm_benh_ten_benh
    on dm_benh (ten_benh);

create table dm_co_quan_bhxh
(
    ma  varchar(20)  not null comment 'Mã cơ quan BHXH'
        primary key,
    ten varchar(200) not null comment 'Tên cơ quan BHXH'
)
    collate = utf8mb4_unicode_ci;

create table dm_dan_toc
(
    id         int                                 not null comment 'Mã ID dân tộc'
        primary key,
    ma         varchar(20)                         not null comment 'Mã dân tộc (01, 02...)',
    ten        varchar(100)                        not null comment 'Tên dân tộc',
    created_at timestamp default CURRENT_TIMESTAMP null,
    updated_at timestamp default CURRENT_TIMESTAMP null on update CURRENT_TIMESTAMP,
    constraint ma
        unique (ma)
)
    comment 'Danh mục Dân tộc' collate = utf8mb4_unicode_ci;

create table dm_doi_tuong
(
    ma   varchar(20)  not null comment 'Mã đối tượng theo BHXH (VD: DN, HT, HC...)'
        primary key,
    ten  varchar(255) not null,
    loai varchar(20)  not null comment 'bhxh, bhyt, ca_hai'
)
    collate = utf8mb4_unicode_ci;

create table dm_doi_tuong_ho_tro
(
    ma                varchar(20)   not null comment 'Mã đối tượng (NGHEO, CANNGHEO, KHAC)'
        primary key,
    ten               varchar(100)  not null,
    ty_le_ho_tro      decimal(5, 2) not null comment 'Tỷ lệ hỗ trợ (%)',
    muc_chuan_ngheo   decimal(15)   null comment 'Mức chuẩn nghèo áp dụng',
    ngay_ap_dung      date          not null,
    ngay_het_hieu_luc date          null,
    ghi_chu           text          null
)
    collate = utf8mb4_unicode_ci;

create table dm_don_vi_hanh_chinh
(
    ma     varchar(20)  not null comment 'Mã đơn vị hành chính theo quy định của Tổng cục Thống kê'
        primary key,
    ten    varchar(255) not null,
    cap    varchar(20)  not null comment 'tinh, huyen, xa',
    ma_cha varchar(20)  null,
    constraint dm_don_vi_hanh_chinh_ibfk_1
        foreign key (ma_cha) references dm_donvi_hanhchinh (ma)
)
    collate = utf8mb4_unicode_ci;

create index ma_cha
    on dm_don_vi_hanh_chinh (ma_cha);

create table dm_donvi_hanhchinh
(
    ma                    varchar(20)                        not null comment 'Mã đơn vị hành chính theo quy định của Tổng cục Thống kê'
        primary key,
    ten                   varchar(255)                       not null,
    cap                   varchar(20)                        null comment 'tinh, huyen, xa',
    ma_cha                varchar(20)                        null,
    ma_dbhc               varchar(20)                        not null comment 'Mã địa bàn hành chính (DBHC)',
    tinh_trang            tinyint                            not null comment '1: Hoạt động, 0: Không hoạt động',
    can_cu                varchar(255)                       null comment 'Căn cứ pháp lý',
    ngay_hieu_luc         varchar(30)                        null comment 'Ngày hiệu lực',
    ngay_het_hieu_luc     varchar(30)                        null comment 'Ngày hết hiệu lực',
    id_dia_ban_hanh_chinh bigint                             null comment 'ID địa bàn hành chính liên quan',
    last_update           datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP,
    constraint uk_donvi_ma_dbhc
        unique (ma_dbhc)
)
    collate = utf8mb4_unicode_ci;

create index idx_donvi_ma_cha
    on dm_donvi_hanhchinh (ma_cha);

create table dm_hanhchinh_coquan_bhxh
(
    id              bigint auto_increment comment 'ID nội bộ'
        primary key,
    stt             int                                null comment 'Số thứ tự hiển thị',
    ma_tinh         varchar(20)                        not null comment 'Mã tỉnh (mapping với dm_donvi_hanhchinh.ma_dbhc)',
    ten_tinh        varchar(255)                       not null comment 'Tên tỉnh',
    ma_coquan_bhxh  varchar(20)                        not null comment 'Mã cơ quan BHXH',
    ten_coquan_bhxh varchar(255)                       not null comment 'Tên cơ quan BHXH',
    ma_xa           varchar(20)                        not null comment 'Mã xã quản lý',
    ten_xa          varchar(255)                       not null comment 'Tên xã quản lý',
    last_update     datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP
)
    comment 'Bảng mapping many-many dm_coquan_bhxh vs dm_donvi_hanhchinh' collate = utf8mb4_unicode_ci;

create index idx_bhxh_ma_tinh
    on dm_hanhchinh_coquan_bhxh (ma_tinh);

create index idx_bhxh_ma_xa
    on dm_hanhchinh_coquan_bhxh (ma_xa);

create table dm_lai_suat
(
    id                bigint auto_increment
        primary key,
    ngay_ap_dung      date          not null,
    ngay_het_hieu_luc date          not null,
    ten               varchar(255)  not null,
    trang_thai        bit           null,
    ty_le             decimal(5, 2) not null
);

create table dm_loai_doi_tuong
(
    id         int                                 not null comment 'Mã ID loại đối tượng'
        primary key,
    ma         varchar(20)                         not null comment 'Mã loại đối tượng (CC, TE...)',
    ten        varchar(255)                        not null comment 'Tên loại đối tượng',
    nhom       varchar(255)                        null comment 'Nhóm đối tượng',
    ma_dt_kcb  varchar(255)                        null comment 'Mã đối tượng KCB',
    ten_dt_kcb varchar(255)                        null comment 'Tên đối tượng KCB',
    created_at timestamp default CURRENT_TIMESTAMP null,
    updated_at timestamp default CURRENT_TIMESTAMP null on update CURRENT_TIMESTAMP,
    constraint ma
        unique (ma)
)
    comment 'Danh mục Loại đối tượng BHYT' collate = utf8mb4_unicode_ci;

create table dm_loai_giay_to
(
    id         int                                 not null comment 'Mã ID loại giấy tờ'
        primary key,
    ma         varchar(20)                         not null comment 'Mã loại giấy tờ',
    ten        varchar(255)                        not null comment 'Tên loại giấy tờ',
    created_at timestamp default CURRENT_TIMESTAMP null,
    updated_at timestamp default CURRENT_TIMESTAMP null on update CURRENT_TIMESTAMP,
    constraint ma
        unique (ma)
)
    comment 'Danh mục Loại giấy tờ tùy thân' collate = utf8mb4_unicode_ci;

create table dm_loai_su_vu
(
    ma             varchar(20)          not null
        primary key,
    ten            varchar(100)         not null,
    loai_hanh_dong varchar(20)          not null comment 'cap_nhat, hoan_tien, giam_dong',
    thoi_gian_sla  int                  not null comment 'Thời gian xử lý SLA (phút)',
    tu_dong_xu_ly  tinyint(1) default 0 null,
    mo_ta          text                 null
)
    collate = utf8mb4_unicode_ci;

create table dm_luong_co_so
(
    id                 int auto_increment
        primary key,
    so_tien            decimal(15, 2)                      not null comment 'Mức lương cơ sở',
    thoi_gian_bat_dau  date                                not null comment 'Thời gian bắt đầu áp dụng',
    thoi_gian_ket_thuc date                                null comment 'Thời gian kết thúc áp dụng',
    created_at         timestamp default CURRENT_TIMESTAMP null,
    updated_at         timestamp default CURRENT_TIMESTAMP null on update CURRENT_TIMESTAMP
)
    comment 'Danh mục Lương cơ sở qua các thời kỳ' collate = utf8mb4_unicode_ci;

create table dm_ma_vung
(
    id         int                                 not null comment 'Mã ID vùng'
        primary key,
    ma         varchar(20)                         not null comment 'Mã vùng (K1, K2...)',
    ten        text                                not null comment 'Tên/Mô tả vùng',
    created_at timestamp default CURRENT_TIMESTAMP null,
    updated_at timestamp default CURRENT_TIMESTAMP null on update CURRENT_TIMESTAMP,
    constraint dm_ma_vung
        unique (ma)
)
    comment 'Danh mục Mã vùng (K1, K2, K3...)' collate = utf8mb4_unicode_ci;

create table dm_muc_huong_bhyt
(
    id                int                                 not null comment 'Mã ID mức hưởng'
        primary key,
    ma                varchar(20)                         not null comment 'Mã mức hưởng (1, 2, 3...)',
    ten               text                                not null comment 'Tên mức hưởng',
    noi_dung          text                                null comment 'Nội dung chi tiết mức hưởng',
    ty_le_dong        decimal(8, 2)                       not null,
    ngay_ap_dung      date                                not null,
    ngay_het_hieu_luc date                                null,
    created_at        timestamp default CURRENT_TIMESTAMP null,
    updated_at        timestamp default CURRENT_TIMESTAMP null on update CURRENT_TIMESTAMP,
    constraint dm_muc_huong_bhyt
        unique (ma),
    constraint ma
        unique (ma)
)
    comment 'Danh mục Mức hưởng BHYT' collate = utf8mb4_unicode_ci;

create table dm_noi_kcb
(
    ma        varchar(20)  not null comment 'Mã nơi KCB theo quy định của BHXH'
        primary key,
    ten       varchar(255) not null,
    ma_tinh   varchar(20)  not null,
    ma_huyen  varchar(20)  not null,
    tuyen_kcb int          not null comment 'Tuyến KCB (1, 2, 3)',
    dia_chi   varchar(500) null,
    constraint dm_noi_kcb_ibfk_1
        foreign key (ma_tinh) references dm_donvi_hanhchinh (ma),
    constraint dm_noi_kcb_ibfk_2
        foreign key (ma_huyen) references dm_donvi_hanhchinh (ma)
)
    collate = utf8mb4_unicode_ci;

create index ma_huyen
    on dm_noi_kcb (ma_huyen);

create index ma_tinh
    on dm_noi_kcb (ma_tinh);

create table dm_phuong_an_dong_bhxh
(
    id         varchar(50)                         not null comment 'Mã ID phương án (TLĐ, TM...)'
        primary key,
    ma         varchar(50)                         not null comment 'Mã phương án',
    ten        varchar(255)                        not null comment 'Tên phương án',
    giai_thich text                                null comment 'Giải thích chi tiết',
    parent_id  varchar(50)                         null comment 'ID cha (nếu có)',
    status     tinyint   default 0                 null comment '1 = active',
    created_at timestamp default CURRENT_TIMESTAMP null,
    updated_at timestamp default CURRENT_TIMESTAMP null on update CURRENT_TIMESTAMP,
    constraint uk_dm_phuong_an_dong_bhxh_ma
        unique (ma)
)
    comment 'Danh mục Loại phương án BHXH tự nguyện' collate = utf8mb4_unicode_ci;

create table dm_phuong_an_dong_bhyt
(
    id         varchar(50)                         not null comment 'Mã ID phương án (TLĐ, TM...)'
        primary key,
    ma         varchar(50)                         not null comment 'Mã phương án',
    ten        varchar(255)                        not null comment 'Tên phương án',
    giai_thich text                                null comment 'Giải thích chi tiết',
    parent_id  varchar(50)                         null comment 'ID cha (nếu có)',
    status     tinyint   default 0                 null comment '1 = active',
    created_at timestamp default CURRENT_TIMESTAMP null,
    updated_at timestamp default CURRENT_TIMESTAMP null on update CURRENT_TIMESTAMP,
    constraint uk_dm_phuong_an_dong_bhyt_ma
        unique (ma)
)
    comment 'Danh mục Phương án đóng BHYT (Tăng mới, Giai hạn...)' collate = utf8mb4_unicode_ci;

create table dm_phuong_thuc_dong
(
    id         int                                 not null comment 'Mã ID phương thức đóng'
        primary key,
    ma         varchar(20)                         not null comment 'Mã phương thức (1, 3, 6, 12, VS, TH...)',
    ten        varchar(255)                        not null comment 'Tên phương thức đóng',
    created_at timestamp default CURRENT_TIMESTAMP null,
    updated_at timestamp default CURRENT_TIMESTAMP null on update CURRENT_TIMESTAMP,
    constraint ma
        unique (ma)
)
    comment 'Danh mục Phương thức đóng BHXH/BHYT' collate = utf8mb4_unicode_ci;

create table dm_quan_he_gia_dinh
(
    id         int                                 not null comment 'Mã ID quan hệ'
        primary key,
    ma         varchar(20)                         not null comment 'Mã quan hệ (00, 01...)',
    ten        varchar(100)                        not null comment 'Tên mối quan hệ với chủ hộ',
    created_at timestamp default CURRENT_TIMESTAMP null,
    updated_at timestamp default CURRENT_TIMESTAMP null on update CURRENT_TIMESTAMP,
    constraint ma
        unique (ma)
)
    comment 'Danh mục Mối quan hệ gia đình' collate = utf8mb4_unicode_ci;

create table dm_quoc_tich
(
    id         int                                 not null comment 'Mã ID quốc tịch'
        primary key,
    ma         varchar(10)                         not null comment 'Mã quốc tịch (VN, US...)',
    ten        varchar(255)                        not null comment 'Tên quốc gia/vùng lãnh thổ',
    created_at timestamp default CURRENT_TIMESTAMP null,
    updated_at timestamp default CURRENT_TIMESTAMP null on update CURRENT_TIMESTAMP,
    constraint ma
        unique (ma)
)
    comment 'Danh mục Quốc tịch' collate = utf8mb4_unicode_ci;

create table dm_ty_le_thu
(
    id                int auto_increment
        primary key,
    ngay_ap_dung      date          not null,
    ngay_het_hieu_luc date          null,
    ty_le_dong        decimal(5, 2) not null
);

create table dm_ty_le_thu_lao
(
    id                      int auto_increment
        primary key,
    loai_ho_so              varchar(20)    not null comment 'bhxh_tu_nguyen, bhyt_ho_gia_dinh',
    muc_doanh_thu_toi_thieu decimal(15, 2) not null,
    ty_le                   decimal(5, 2)  not null,
    ngay_ap_dung            date           not null,
    ghi_chu                 text           null
)
    collate = utf8mb4_unicode_ci;

create table flyway_schema_history
(
    installed_rank int                                 not null
        primary key,
    version        varchar(50)                         null,
    description    varchar(200)                        not null,
    type           varchar(20)                         not null,
    script         varchar(1000)                       not null,
    checksum       int                                 null,
    installed_by   varchar(100)                        not null,
    installed_on   timestamp default CURRENT_TIMESTAMP not null,
    execution_time int                                 not null,
    success        tinyint(1)                          not null
);

create index flyway_schema_history_s_idx
    on flyway_schema_history (success);

-- --------------------------------------------------------------------------------------
-- SECTION 2: SYSTEM TABLES (HỆ THỐNG)
-- --------------------------------------------------------------------------------------

CREATE TABLE IF NOT EXISTS agency
(
    id                   int auto_increment
        primary key,
    name                 varchar(255)   null,
    ascii_name           varchar(255)   null,
    description          varchar(255)   null,
    email                varchar(255)   null,
    phone                varchar(20)    null,
    region_id            int            null,
    city_id              int            null,
    district_id          int            null,
    ward_id              int            null,
    address              varchar(255)   null,
    status               int default 10 null,
    parent_id            int            null,
    admin_id             int            null,
    level                int default 1  null,
    path                 varchar(255)   null,
    created_at           int            null,
    updated_at           int            null,
    created_by           int            null,
    updated_by           int            null,
    agency_code          varchar(255)   null comment 'Mã đại lý',
    agent_level_id       int            null comment 'Cấp đại lý',
    social_insurance     varchar(255)   null comment 'Cơ quan BHXH',
    bank_account         varchar(255)   null comment 'Số tài khoản',
    tax_code             varchar(255)   null comment 'Mã số thuế',
    username             varchar(255)   null comment 'Tên tài khoản',
    fullname             varchar(255)   null,
    representative_name  varchar(255)   null comment 'Tên đại diện',
    representative_phone varchar(255)   null comment 'Số điện thoại đại diện',
    representative_email varchar(255)   null comment 'Email đại diện',
    bank_name            varchar(255)   null comment 'Tên ngân hàng',
    bank_owner           varchar(255)   null comment 'Chủ tài khoản',
    bank_city_id         int            null comment 'Tỉnh/Thành phố mở tài khoản'
);

CREATE TABLE IF NOT EXISTS seller
(
    id                  int auto_increment
        primary key,
    code                varchar(45)   null,
    agency_name         varchar(255)  null,
    contract_code       varchar(100)  null,
    full_name           varchar(255)  null,
    ascii_name          varchar(255)  null,
    phone_number        varchar(64)   null,
    email               varchar(128)  null,
    address             varchar(512)  null,
    city_id             int           null,
    district_id         int           null,
    ward_id             int           null,
    street_id           int           null,
    cccd                varchar(45)   null,
    birthday            int           null,
    sex                 int           null,
    type                int           null,
    username            varchar(128)  null,
    password_hash       varchar(512)  null,
    status              int           null,
    created_at          int           null,
    updated_at          int           null,
    channel             int           null,
    vdcp_id             varchar(128)  null,
    bank_account        text          null comment 'Mảng json thông tin chuyển khoản',
    contract_images     text          null comment ';image1;image2;image3;',
    user_id             int           null,
    nv_index            int default 0 null,
    agency_id           int           null,
    bank_code           text          null comment 'Ngân hàng',
    account_holder_name text          null comment 'Chủ tài khoản'
);

-- --------------------------------------------------------------------------------------
-- SECTION 3: BUSINESS TABLES (NGHIỆP VỤ)
-- --------------------------------------------------------------------------------------

-- 3.1. NGƯỜI THAM GIA
CREATE TABLE IF NOT EXISTS nv_nguoi_tham_gia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ma_so_bhxh VARCHAR(20) NULL COMMENT 'Mã số BHXH do cơ quan BHXH cấp',
    ho_ten VARCHAR(100) NOT NULL,
    gioi_tinh VARCHAR(10) NOT NULL COMMENT 'nam, nu',
    ngay_sinh DATE NOT NULL,
    so_cccd VARCHAR(20) NULL,
    ma_quoc_tich VARCHAR(10) DEFAULT 'VN' NOT NULL,
    ma_dan_toc VARCHAR(20) DEFAULT '01' NOT NULL,
    so_dien_thoai VARCHAR(20) NULL,
    email VARCHAR(100) NULL,
    ma_tinh_tt VARCHAR(20) NULL COMMENT 'Tỉnh thường trú',
    ma_xa_tt VARCHAR(20) NULL COMMENT 'Xã thường trú',
    chi_tiet_noi_tt VARCHAR(200) NULL COMMENT 'Chi tiết địa chỉ thường trú',
    ma_tinh_ks VARCHAR(20) NULL COMMENT 'Tỉnh khai sinh',
    ma_xa_ks VARCHAR(20) NULL COMMENT 'Xã khai sinh',
    chi_tiet_noi_ks VARCHAR(200) NULL COMMENT 'Chi tiết địa chỉ khai sinh',
    ngay_tao DATETIME DEFAULT CURRENT_TIMESTAMP NULL,
    ngay_cap_nhat DATETIME DEFAULT CURRENT_TIMESTAMP NULL ON UPDATE CURRENT_TIMESTAMP,
    mo_rong JSON NULL COMMENT 'Thông tin mở rộng như họ tên cha/mẹ/giám hộ',
    -- UNIQUE KEY ma_so_bhxh (ma_so_bhxh),
    FOREIGN KEY (ma_quoc_tich) REFERENCES dm_quoc_tich (ma),
    FOREIGN KEY (ma_dan_toc) REFERENCES dm_dan_toc (ma),
    FOREIGN KEY (ma_tinh_tt) REFERENCES dm_donvi_hanhchinh (ma_dbhc),
    FOREIGN KEY (ma_xa_tt) REFERENCES dm_donvi_hanhchinh (ma_dbhc),
    FOREIGN KEY (ma_tinh_ks) REFERENCES dm_donvi_hanhchinh (ma_dbhc),
    FOREIGN KEY (ma_xa_ks) REFERENCES dm_donvi_hanhchinh (ma_dbhc)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Quản lý thông tin người tham gia';

-- 3.2. HỒ SƠ ĐĂNG KÝ
CREATE TABLE IF NOT EXISTS nv_ho_so_dang_ky (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ma_ho_so VARCHAR(50) NOT NULL,
    ngay_lap DATE NOT NULL,
    trang_thai VARCHAR(20) NOT NULL DEFAULT 'moi_tao',
    loai_nghiep_vu VARCHAR(50) NOT NULL COMMENT 'bhxh_tu_nguyen, bhyt_ho_gia_dinh, ca_hai',
    nhan_vien_thu_id INT NULL COMMENT 'ID nhân viên thu (NULL nếu chưa xác thực)',
    dai_ly_thu_id INT NULL COMMENT 'ID đại lý thu (NULL nếu chưa xác thực)',
    nguoi_tham_gia_id INT NOT NULL,
    ma_tham_chieu_ivan VARCHAR(100) NULL COMMENT 'Mã hồ sơ tham chiếu trên hệ thống IVAN',
    ngay_gui_ivan DATETIME NULL,
    ket_qua_ivan JSON NULL COMMENT 'Kết quả trả về từ hệ thống IVAN',
    ngay_tao DATETIME DEFAULT CURRENT_TIMESTAMP NULL,
    ngay_cap_nhat DATETIME DEFAULT CURRENT_TIMESTAMP NULL ON UPDATE CURRENT_TIMESTAMP,
    mo_rong JSON NULL COMMENT 'Thông tin mở rộng cho từng loại nghiệp vụ',
    bien_lai_id  INT NULL,
    su_vu_id  INT NULL,
    da_hoan_tien        tinyint(1)                            null comment 'Đã hoàn tiền chưa (0= chưa hoàn tiền, 1= đã hoàn tiền)',
    so_tien_hoso        decimal(15, 2)                        null comment 'Final profile amount (denormalized for performance)',
    trang_thai_bien_lai varchar(20) default 'CAN_TAO_MOI'     not null comment 'trang thái biên lai (CAN_TAO_MOI, CAN_THAY_THE, HOP_LE) để quyết định tạo/thay thế biên lai',
    UNIQUE KEY ma_ho_so (ma_ho_so),
    FOREIGN KEY (nguoi_tham_gia_id) REFERENCES nv_nguoi_tham_gia (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Quản lý hồ sơ đăng ký';

-- 3.4. CHI TIẾT BHXH TỰ NGUYỆN
CREATE TABLE IF NOT EXISTS nv_chi_tiet_bhxh (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ho_so_id INT NOT NULL,
    ma_co_quan_bhxh VARCHAR(100) NULL COMMENT 'Mã cơ quan BHXH',
    ma_phuong_an_dong VARCHAR(20) NULL COMMENT 'Mã phương án đóng: dang_ky_moi, dong_tiep, dieu_chinh',
    muc_dong DECIMAL(15) NOT NULL COMMENT 'Mức thu nhập tháng làm căn cứ đóng',
    ma_phuong_thuc_dong VARCHAR(20) NOT NULL,
    tu_ngay DATE NULL COMMENT 'Từ ngày đóng',
    den_ngay DATE NULL COMMENT 'Đến ngày đóng',
    so_thang_dong INT NOT NULL,
    tien_dong DECIMAL(15, 2) NOT NULL,
    ty_le_ho_tro_nsnn   decimal(5, 2) NULL comment 'Tỷ lệ hỗ trợ NSNN (%)',
    tien_ho_tro_nsnn DECIMAL(15, 2) DEFAULT 0.00 NULL,
    ty_le_ho_tro_nsdp   decimal(5, 2)               null comment 'Tỷ lệ hỗ trợ NSĐP (%)',
    tien_ho_tro_nsdp DECIMAL(15, 2) DEFAULT 0.00 NOT NULL COMMENT 'Tiền hỗ trợ NSĐP (nếu có)',
    tong_tien_dong DECIMAL(15, 2) NULL COMMENT 'Tổng tiền phải đóng',
    ma_doi_tuong_ho_tro VARCHAR(20) NULL COMMENT 'Mã đối tượng hỗ trợ',
    ma_mau_d05_ts VARCHAR(50) NULL COMMENT 'Mã tham chiếu đến mẫu D05-TS',
    ghi_chu       VARCHAR(2000) NULL,
    UNIQUE KEY uk_ho_so_id (ho_so_id),
    FOREIGN KEY (ho_so_id) REFERENCES nv_ho_so_dang_ky (id),
    FOREIGN KEY (ma_co_quan_bhxh) REFERENCES dm_co_quan_bhxh (ma),
    FOREIGN KEY (ma_phuong_an_dong) REFERENCES dm_phuong_an_dong_bhxh (ma),
    FOREIGN KEY (ma_phuong_thuc_dong) REFERENCES dm_phuong_thuc_dong (ma),
    FOREIGN KEY (ma_doi_tuong_ho_tro) REFERENCES dm_doi_tuong_ho_tro (ma)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Chi tiết BHXH';

-- 3.5. CHI TIẾT BHYT HỘ GIA ĐÌNH
CREATE TABLE IF NOT EXISTS nv_chi_tiet_bhyt (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ho_so_id INT NOT NULL,
    ma_co_quan_bhxh VARCHAR(20) NULL COMMENT 'Mã cơ quan BHXH',
    ma_noi_kcb VARCHAR(20) NOT NULL,
    tu_ngay DATE NOT NULL,
    den_ngay DATE NOT NULL,
    ma_mau_d03_ts VARCHAR(50) NULL COMMENT 'Mã tham chiếu đến mẫu D03-TS',
    ma_tinh_kham_benh VARCHAR(20) NULL COMMENT 'Mã tỉnh/thành phố đăng ký KCB (Step 2)',
    ma_benh_vien VARCHAR(20) NULL COMMENT 'Mã bệnh viện (có thể lưu trong ma_noi_kcb)',
    ma_phuong_an_dong VARCHAR(20) NULL COMMENT 'Mã phương án đóng: tang_moi, gia_han (snapshot từ Step 1)',
    so_thang_dong INT NULL COMMENT 'Số tháng đóng: 3, 6, 12 (Step 2)',
    ty_le_ho_gia_dinh DECIMAL(5, 2) NULL COMMENT 'Tỷ lệ đóng hộ gia đình: 100, 70, 60, 50, 40 (snapshot Step 3)',
    ty_le_ho_tro_nsnn decimal(5, 2)               null comment 'Tỷ lệ hỗ trợ NSNN (%)',
    tien_ho_tro_nsnn DECIMAL(15, 2) DEFAULT 0.00 NULL COMMENT 'Tiền hỗ trợ NSNN (snapshot Step 3)',
    ty_le_ho_tro_nsdp decimal(5, 2)               null comment 'Tỷ lệ hỗ trợ NSĐP (%)',
    tien_ho_tro_nsdp DECIMAL(15, 2) DEFAULT 0.00 NULL COMMENT 'Tiền hỗ trợ NSĐP (snapshot Step 3)',
    tong_tien_dong DECIMAL(15, 2) NULL COMMENT 'Tổng tiền NTG phải đóng (snapshot Step 3)',
    ma_muc_huong_bhyt varchar(20)                 null comment 'Mã mức hưởng BHYT',
    ghi_chu           varchar(2000)               null,
    ma_loai_doi_tuong_bhyt varchar(50)            null,

    constraint fk_nv_chi_tiet_bhyt_ma_loai_dt FOREIGN KEY (ma_loai_doi_tuong_bhyt) REFERENCES dm_loai_doi_tuong_bhyt (ma_loai_doi_tuong),
    FOREIGN KEY (ma_co_quan_bhxh) REFERENCES dm_co_quan_bhxh (ma),
    FOREIGN KEY (ma_phuong_an_dong) REFERENCES dm_phuong_an_dong_bhyt (ma),
    FOREIGN KEY (ma_tinh_kham_benh) REFERENCES dm_donvi_hanhchinh (ma_dbhc),
    FOREIGN KEY (ho_so_id) REFERENCES nv_ho_so_dang_ky (id),
    FOREIGN KEY (ma_noi_kcb) REFERENCES dm_coso_kcb (ma),
    FOREIGN KEY (ma_muc_huong_bhyt) REFERENCES dm_muc_huong_bhyt (ma)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Chi tiết BHYT';

-- 3.9. THÀNH VIÊN HỘ GIA ĐÌNH
CREATE TABLE IF NOT EXISTS nv_thanh_vien_ho_gia_dinh (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ho_so_id INT NOT NULL COMMENT 'ID hồ sơ (QUAN HỆ TRỰC TIẾP - DUY NHẤT)',
    ma_ho_gia_dinh VARCHAR(50) NULL COMMENT 'Mã hộ gia đình theo hệ thống BHXH',
    ho_ten VARCHAR(100) NOT NULL,
    gioi_tinh VARCHAR(10) NOT NULL COMMENT 'nam, nu',
    ngay_sinh DATE NOT NULL,
    so_cccd VARCHAR(20) NULL,
    ma_quoc_tich VARCHAR(10) NULL,
    ma_dan_toc VARCHAR(20) NULL,
    so_dien_thoai VARCHAR(20) NULL,
    email VARCHAR(100) NULL,
    ma_quan_he VARCHAR(20) NOT NULL,
    stt_khai_bao INT NULL COMMENT 'Thứ tự khai báo để tính mức đóng',
    is_nguoi_tham_gia TINYINT(1) DEFAULT 0 NOT NULL COMMENT '1 = người tham gia chính, 0 = thành viên khác',
    ma_so_bhxh VARCHAR(20) NULL,
    ty_le_dong DECIMAL(5, 2) NULL COMMENT 'Tỷ lệ % đóng (100%, 70%, 60%...) - dùng cho BHYT',
    so_tien_dong DECIMAL(15, 2) NULL COMMENT 'Số tiền phải đóng - dùng cho BHYT',
    so_thang_dong INT NULL COMMENT 'Số tháng đóng - dùng cho BHYT',
    ma_tinh_ks VARCHAR(20) NULL COMMENT 'Tỉnh khai sinh',
    ma_xa_ks VARCHAR(20) NULL COMMENT 'Xã khai sinh',
    ma_tinh_tt        VARCHAR(20)   NULL comment 'Mã Tỉnh thường trú',
    ma_xa_tt          VARCHAR(20)   NULL comment 'Mã Xã thường trú',
    chi_tiet_noi_tt   VARCHAR(255)  NULL,
    chi_tiet_noi_ks VARCHAR(200) NULL COMMENT 'Chi tiết địa chỉ khai sinh',
    ngay_tao DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL,
    ngay_cap_nhat DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (ho_so_id) REFERENCES nv_ho_so_dang_ky (id) ON DELETE CASCADE,
    FOREIGN KEY (ma_quoc_tich) REFERENCES dm_quoc_tich (ma),
    FOREIGN KEY (ma_dan_toc) REFERENCES dm_dan_toc (ma),
    FOREIGN KEY (ma_quan_he) REFERENCES dm_quan_he_gia_dinh (ma),
    FOREIGN KEY (ma_tinh_ks) REFERENCES dm_donvi_hanhchinh (ma_dbhc),
    FOREIGN KEY (ma_xa_ks) REFERENCES dm_donvi_hanhchinh (ma_dbhc),
    FOREIGN KEY (ma_tinh_tt) REFERENCES dm_donvi_hanhchinh (ma_dbhc),
    FOREIGN KEY (ma_xa_tt) REFERENCES dm_donvi_hanhchinh (ma_dbhc)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Thành viên hộ gia đình';

-- 3.10. SỰ VỤ
CREATE TABLE IF NOT EXISTS nv_su_vu
(
    id                  int auto_increment
    primary key,
    ma_su_vu            varchar(50)                           not null,
    ho_so_id            int                                   not null,
    ma_loai_su_vu       varchar(20)                           not null,
    noi_dung            text                                  not null,
    dulieu_cu           json                                  null comment 'Dữ liệu cũ trước khi thay đổi',
    dulieu_moi          json                                  null comment 'Dữ liệu mới đề xuất thay đổi',
    changed_fields      json                                  null comment 'Chi tiết các trường thay đổi: {"field": {"old": X, "new": Y}}',
    chi_phi_ban_dau     decimal(15, 2)                        null comment 'Số tiền phải đóng TRƯỚC khi cập nhật',
    chi_phi_cap_nhat    decimal(15, 2)                        null comment 'Số tiền phải đóng SAU khi cập nhật',
    version             int         default 1                 not null comment 'Phiên bản sự vụ (1, 2, 3..)',
    parent_su_vu_id     int                                   null comment 'Link đến phiên bản trước (nếu có)',
    root_su_vu_id       int                                   null comment 'Link đến phiên bản đầu tiên',
    event_type          varchar(50) default 'CREATED'         null comment 'CREATED, UPDATED, APPROVED, REJECTED',
    is_latest           tinyint(1)  default 1                 null comment '1=phiên bản hiện tại, 0=đã bị supersede',
    trang_thai          varchar(20) default 'moi_tao'         null comment 'moi_tao, dang_xu_ly, da_giai_quyet, da_huy',
    nguoi_tao_id        int                                   not null,
    nguoi_xu_ly_id      int                                   null,
    ngay_tao            datetime    default CURRENT_TIMESTAMP null,
    ngay_cap_nhat       datetime                              null comment 'Thời điểm cập nhật gần nhất',
    ngay_giai_quyet     datetime                              null,
    ket_qua             text                                  null comment 'Kết quả giải quyết sự vụ',
    loai_nghiep_vu      varchar(20)                           not null,
    ten_nguoi_tham_gia  varchar(100)                          null comment 'Tên người tham gia',
    ma_bhxh             varchar(20)                           null comment 'Mã bhxh',
    nguoi_tao           varchar(100)                          null comment 'Tên người tạo sự vụ',
    nguoi_xu_ly         varchar(100)                          null comment 'Tên người xử lý sụ vụ',
    deleted_at          datetime                              null comment 'Thời điểm xoá sự vụ',
    constraint ma_su_vu
    unique (ma_su_vu),
    constraint nv_su_vu_ibfk_1
    foreign key (ho_so_id) references nv_ho_so_dang_ky (id)
)    comment 'Quản lý sự vụ' collate = utf8mb4_unicode_ci;

create index ho_so_id
    on nv_su_vu (ho_so_id);

create index ma_loai_su_vu
    on nv_su_vu (ma_loai_su_vu);

-- --------------------------------------------------------------------------------------
-- ADD TABLE: nv_bien_lai
-- --------------------------------------------------------------------------------------
CREATE TABLE IF NOT EXISTS nv_bien_lai (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ma_bien_lai VARCHAR(50) NULL COMMENT 'Mã biên lai',
    so_bien_lai VARCHAR(50) NULL COMMENT 'Số biên lai',
    ho_so_id INT NOT NULL COMMENT 'ID hồ sơ liên quan',
    nguoi_tham_gia_id INT NOT NULL COMMENT 'ID người tham gia',
    nguoi_nop VARCHAR(50) NULL COMMENT 'Tên người nộp',
    loai_bien_lai VARCHAR(20) DEFAULT 'c45' NULL COMMENT 'c45, hoan_tra',
    noi_dung_thu TEXT NULL,
    so_thang_dong INT NULL COMMENT 'Số tháng đóng',
    so_tien DECIMAL(15, 2) NOT NULL COMMENT 'Số tiền cần thanh toán',
    han_thanh_toan DATETIME NULL COMMENT 'Hạn thanh toán',
    ngay_thanh_toan DATETIME NULL COMMENT 'Ngày thực tế thanh toán (nếu đã thanh toán)',

    trang_thai VARCHAR(20) DEFAULT 'da_lap' NOT NULL COMMENT 'da_lap, da_thu, da_huy',
    ngay_lap DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL COMMENT 'Ngày lập biên lai',
    ngay_huy DATETIME NULL COMMENT 'Ngày huỷ biên lai',
    file_bien_lai VARCHAR(1000) NULL COMMENT 'Đường dẫn file hoá đơn điện tử',
    trang_thai_xuat_bien_lai VARCHAR(1000) NULL COMMENT 'lưu resp (thành công, thất bại mã lỗi) xuất biên lai phía Ivoice',
    dai_ly_id INT NULL COMMENT 'ID đại lý',
    dai_ly VARCHAR(255) NULL COMMENT 'Tên đại lý',
    nguoi_thu_id INT  NULL COMMENT 'ID seller',
    nguoi_thu VARCHAR(255) NULL COMMENT 'Tên seller',
    UNIQUE KEY ma_hoa_don (ma_bien_lai),
    FOREIGN KEY (ho_so_id) REFERENCES nv_ho_so_dang_ky (id)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Quản lý biên lai thanh toán';

-- --------------------------------------------------------------------------------------
-- ADD TABLE: nv_tai_lieu_dinh_kem
-- --------------------------------------------------------------------------------------
create table if not exists nv_tai_lieu_dinh_kem
(
    id            int auto_increment
    primary key,
    ho_so_id      int                                   null comment 'ID hồ sơ (nullable cho tài liệu chưa gắn)',
    loai_tai_lieu varchar(50)                           not null comment 'cccd_mtr, cccd_msv, giay_khai_sinh, ho_so_khac',
    duong_dan     varchar(500)                          not null comment 'Đường dẫn file trên server',
    ten_file      varchar(255)                          not null,
    kieu_file     varchar(50)                           not null comment 'image/jpeg, application/pdf...',
    kich_thuoc    decimal                               not null comment 'Kích thước file (bytes)',
    trang_thai    varchar(20) default 'hoat_dong'       not null comment 'chua_gan_ho_so, hoat_dong, da_xoa',
    ngay_tao      datetime    default CURRENT_TIMESTAMP null,
    constraint nv_tai_lieu_dinh_kem_ibfk_1
    foreign key (ho_so_id) references nv_ho_so_dang_ky (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Tài liệu đính kèm';

-- --------------------------------------------------------------------------------------
-- CREATE TABLE: nv_lich_su_ho_so
-- --------------------------------------------------------------------------------------

CREATE TABLE IF NOT EXISTS nv_lich_su_ho_so
(
    id                     BIGINT AUTO_INCREMENT PRIMARY KEY,
    ho_so_id               BIGINT      NOT NULL COMMENT 'ID hồ sơ gốc',
    loai_nghiep_vu         VARCHAR(50) NOT NULL COMMENT 'bhxh_tu_nguyen, bhyt_ho_gia_dinh, ca_hai',
    trang_thai             VARCHAR(50) NULL COMMENT 'Trạng thái hồ sơ tại thời điểm log (có thể NULL nếu log không liên quan trạng thái)',
    loai_log               VARCHAR(20) NOT NULL COMMENT 'Phân loại log: EVENT, KHAC',
    lich_su_ho_so_truoc_id BIGINT NULL COMMENT 'ID bản ghi lịch sử hồ sơ trước đó',
    nhan_vien_thu_id       BIGINT NULL COMMENT 'ID nhân viên thu (seller)',
    ten_nhan_vien_thu      VARCHAR(255) NULL COMMENT 'Tên nhân viên thu',
    dai_ly_thu_id          BIGINT NULL COMMENT 'ID đại lý thu',
    ten_dai_ly_thu         VARCHAR(255) NULL COMMENT 'Tên đại lý thu',
    ngay_thuc_hien         DATETIME    NOT NULL COMMENT 'Thời điểm thực hiện nghiệp vụ',
    noi_dung               TEXT NULL COMMENT 'Nội dung log',
    ghi_chu                TEXT NULL COMMENT 'Ghi chú bổ sung',
    nguoi_tao_id           BIGINT NULL COMMENT 'ID user tạo log',
    ngay_tao               DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Thời điểm tạo bản ghi',
    nguoi_tao              varchar(200)                       null comment 'Tên người tạo action',
    thao_tac               varchar(20)                        null comment 'Tên thao tác',
    su_vu_id               int                                null comment 'id sự vụ',
    ma_su_vu               varchar(50)                        null comment 'Mã sự vụ',
    dot_ke_khai_id         BIGINT      NULL COMMENT 'Id nghiệp vụ đợt kê khai',
    ma_dot_ke_khai         varchar(50) NULL COMMENT 'mã đợt kê khai theo format so thu tu trong thang + thang nam tao',

    INDEX                  idx_ls_ho_so_id (ho_so_id),
    INDEX                  idx_ls_ngay_thuc_hien (ngay_thuc_hien),
    INDEX                  idx_ls_loai_log (loai_log)
) COMMENT='Lịch sử thay đổi hồ sơ';

-- --------------------------------------------------------------------------------------
-- CREATE TABLE: nv_dot_ke_khai
-- --------------------------------------------------------------------------------------
CREATE TABLE nv_dot_ke_khai (
    id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT 'ID đợt kê khai (Primary Key)',
    ma_giao_dich_cq_bhxh VARCHAR(100) UNIQUE NULL COMMENT 'Mã giao dịch với CQ BHXH (nhận từ iVan sau khi nộp)',
    loai_thu_tuc VARCHAR(10) NOT NULL COMMENT 'Loại thủ tục: 602-BHXH Tự nguyện, 603-BHYT Hộ gia đình',
    loai_nghiep_vu VARCHAR(50) NOT NULL COMMENT 'Loại nghiệp vụ: bhxh_tu_nguyen hoặc bhyt_ho_gia_dinh',
    ma_doi_tuong VARCHAR(20) NULL COMMENT 'Mã đối tượng BHYT (chỉ áp dụng cho loại thủ tục 603)',
    ty_le_nsnn_ho_tro DECIMAL(5,2) NULL COMMENT 'Tỷ lệ NSNN hỗ trợ (%) - chỉ áp dụng cho loại thủ tục 603',
    trang_thai VARCHAR(50) NOT NULL COMMENT 'Trạng thái: DANG_GUI, CQ_BHXH_DANG_XU_LY, THANH_CONG, THAT_BAI',
    mo_ta TEXT NULL COMMENT 'Mô tả trạng thái hoặc thông báo lỗi từ iVan',
    tong_so_ho_so INT NOT NULL COMMENT 'Tổng số hồ sơ trong đợt kê khai',
    tong_tien DECIMAL(15,0) NOT NULL COMMENT 'Tổng số tiền của các hồ sơ trong đợt (VND)',
    thoi_gian_gui DATETIME NULL COMMENT 'Thời gian gửi sang CQ BHXH (qua iVan)',
    dai_ly_id BIGINT NOT NULL COMMENT 'ID đại lý thu hộ (FK: ht_dai_ly_thu_ho)',
    nguoi_tao_id BIGINT NOT NULL COMMENT 'ID người tạo đợt kê khai (FK: ht_user)',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Thời gian tạo bản ghi',
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Thời gian cập nhật cuối',
    created_by VARCHAR(100) NULL COMMENT 'Người tạo bản ghi',
    updated_by VARCHAR(100) NULL COMMENT 'Người cập nhật cuối',
    ket_qua_chi_tiet     json             null comment 'Thông tin phản hồi chi tiết từ iVan',
    so_ho_so             varchar(50)      null comment 'định danh đợt kê khai phía BHXH',
    so_thu_tu            int              null comment 'Số thứ tự trong 1 khoảng thời gian quy định (tháng/năm)',

    INDEX idx_dai_ly_trang_thai (dai_ly_id, trang_thai),
    INDEX idx_thoi_gian_gui (thoi_gian_gui),
    INDEX idx_loai_thu_tuc (loai_thu_tuc),

    CONSTRAINT fk_dot_ke_khai_dai_ly
        FOREIGN KEY (ma_doi_tuong) REFERENCES dm_loai_doi_tuong_bhyt(ma_loai_doi_tuong),

    CONSTRAINT chk_tong_tien CHECK (tong_tien >= 0),
    CONSTRAINT chk_tong_so_ho_so CHECK (tong_so_ho_so >= 1)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Đợt kê khai nộp hồ sơ sang CQ BHXH';

-- --------------------------------------------------------------------------------------
-- CREATE TABLE: nv_dot_ke_khai_ho_so
-- --------------------------------------------------------------------------------------
CREATE TABLE nv_dot_ke_khai_ho_so (
    dot_ke_khai_id BIGINT NOT NULL COMMENT 'ID đợt kê khai (FK: nv_dot_ke_khai)',
    ho_so_id INT NOT NULL COMMENT 'ID hồ sơ đăng ký (FK: nv_ho_so_dang_ky)',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Thời gian thêm hồ sơ vào đợt',

    PRIMARY KEY (dot_ke_khai_id, ho_so_id),
    INDEX idx_ho_so_id (ho_so_id),

    CONSTRAINT fk_dkk_hs_dot_ke_khai
        FOREIGN KEY (dot_ke_khai_id) REFERENCES nv_dot_ke_khai(id),
    CONSTRAINT fk_dkk_hs_ho_so
        FOREIGN KEY (ho_so_id) REFERENCES nv_ho_so_dang_ky(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Bảng mapping đợt kê khai và hồ sơ (many-to-many)';

-- --------------------------------------------------------------------------------------
-- CREATE TABLE: nv_to_khai_dien_tu
-- --------------------------------------------------------------------------------------
CREATE TABLE nv_to_khai_dien_tu (
    id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT 'ID tờ khai điện tử (Primary Key)',
    dot_ke_khai_id BIGINT NULL COMMENT 'ID đợt kê khai (FK: nv_dot_ke_khai)',
    ho_so_id INT NULL comment 'ID hồ sơ đăng ký (FK: nv_ho_so_dang_ky) - Mỗi tờ khai gắn với một hồ sơ cụ thể',
    loai_to_khai VARCHAR(20) NOT NULL COMMENT 'Loại tờ khai: D03-TS, D05-TS, TK01-TS',
    ten_file VARCHAR(255) NOT NULL COMMENT 'Tên file tờ khai (ví dụ: D05-TS.xlsx)',
    duong_dan_file VARCHAR(500) NULL COMMENT 'Đường dẫn file trong storage',
    kich_thuoc_file BIGINT NULL COMMENT 'Kích thước file (bytes)',
    da_ky_so BOOLEAN NOT NULL DEFAULT FALSE COMMENT 'Đã ký số chưa (true/false)',
    thoi_gian_ky_so DATETIME NULL COMMENT 'Thời gian thực hiện ký số',
    chu_ky_so TEXT NULL COMMENT 'Chữ ký số (base64 hoặc reference)',
    so_luong_ban_ghi INT NULL COMMENT 'Số lượng bản ghi trong tờ khai (với TK01-TS: số file trong zip)',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'Thời gian tạo bản ghi',
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Thời gian cập nhật cuối',
    created_by VARCHAR(100) NULL COMMENT 'Người tạo bản ghi',
    updated_by VARCHAR(100) NULL COMMENT 'Người cập nhật cuối',
    ket_qua_chi_tiet json NULL COMMENT 'Thông tin phản hồi chi tiết từ iVan',

    INDEX idx_dot_ke_khai_loai (dot_ke_khai_id, loai_to_khai),
    INDEX idx_da_ky_so (da_ky_so),

    constraint fk_to_khai_ho_so
        foreign key (ho_so_id) references nv_ho_so_dang_ky (id)
            on delete cascade
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Tờ khai điện tử (D03-TS, D05-TS, TK01-TS)';

-- NEW TABLE: Financial transactions
CREATE TABLE nv_giao_dich
(
    id                 INT AUTO_INCREMENT PRIMARY KEY,
    ma_giao_dich       VARCHAR(20)    NOT NULL UNIQUE COMMENT 'GD-{6 digits}',
    ho_so_id           INT            NOT NULL,
    loai_giao_dich     VARCHAR(20)    NOT NULL COMMENT 'THU_MOI, TRUY_THU, HOAN_TIEN',
    so_tien            DECIMAL(15, 2) NOT NULL,
    phuong_thuc        VARCHAR(20)    NOT NULL COMMENT 'TIEN_MAT, VI_DIEN_TU, CHUYEN_KHOAN_QR',
    su_vu_id           INT NULL COMMENT 'Incident ID if triggered by incident',
    bien_lai_id        INT NULL COMMENT 'Receipt ID if available',
    loai_tai_khoan     VARCHAR(20)    NOT NULL COMMENT 'SYSADMIN, SELLER, AGENCY',
    nguoi_thuc_hien_id INT            NOT NULL,
    ngay_giao_dich     DATETIME       NOT NULL,
    ghi_chu            TEXT NULL,
    created_at         DATETIME DEFAULT CURRENT_TIMESTAMP,
    created_by         INT NULL,
    FOREIGN KEY (ho_so_id) REFERENCES nv_ho_so_dang_ky (id),
    FOREIGN KEY (su_vu_id) REFERENCES nv_su_vu (id),
    FOREIGN KEY (bien_lai_id) REFERENCES nv_bien_lai (id),
    INDEX              idx_giao_dich_ho_so (ho_so_id, loai_giao_dich, so_tien),
    INDEX              idx_giao_dich_ngay (ngay_giao_dich),
    INDEX              idx_giao_dich_ma (ma_giao_dich)
) COMMENT='Bảng lưu thông tin các giao dịch tài chính';

-- V3: update table and init table phuc vu nghiep vu doi soat

-- them ma vung cho bang hanh chinh
alter table dm_donvi_hanhchinh
    add column ma_vung VARCHAR(20) null comment 'mã vùng phục vụ cho phần chi trả thù lao (I, II, III)';

update dm_donvi_hanhchinh
set ma_vung = 'I'
where ten in ('Thành phố Hà Nội', 'Thành phố Hồ Chí Minh', 'Thành phố Hải Phòng', 'Thành phố Đà Nẵng', 'Tỉnh Đồng Nai');

-- 13
update dm_donvi_hanhchinh
set ma_vung = 'III'
where ten in ('Tỉnh Sơn La', 'Tỉnh Điện Biên', 'Tỉnh Lai Châu', 'Tỉnh Lào Cai', 'Tỉnh Tuyên Quang', 'Tỉnh Cao Bằng', 'Tỉnh Lạng Sơn', 'Tỉnh Quảng Ngãi', 'Tỉnh Gia Lai', 'Tỉnh Đắk Lắk', 'Tỉnh An Giang', 'Tỉnh Cà Mau', 'Tỉnh Đồng Tháp');

update dm_donvi_hanhchinh
set ma_vung = 'II'
where cap = 'tinh' and ma_vung is null;

-- them nv_chinh_sach_thu_lao
CREATE TABLE nv_chinh_sach_thu_lao
(
    id               INT AUTO_INCREMENT PRIMARY KEY,
    loai_nghiep_vu   VARCHAR(20)   NOT NULL COMMENT 'BHYT_HO_GIA_DINH/BHXH_TU_NGUYEN',
    loai_san_pham    VARCHAR(20)   NULL COMMENT 'bhxh - BHXH tự nguyện; bhyt mapping 1 - 1 mã loại đối tượng của bảo hiểm y tế',
    phuong_thuc_dong VARCHAR(20)   NULL COMMENT 'bảng dm_phuong_thuc_dong',
    vung_dvhc        VARCHAR(20)   NOT NULL COMMENT 'mã vùng tương ứng với nơi cư trú người tham gia',
    phuong_an_dong   VARCHAR(20)   NOT NULL COMMENT 'Tang moi (TM) hoac khac tang moi',
    trang_thai       TINYINT       NOT NULL DEFAULT 1 COMMENT '1: có hiệu lực - 0: hết hiệu lực',
    ti_le_thu_lao    DECIMAL(5, 2) NOT NULL COMMENT 'ti_le_thu_lao',
    ghi_chu          TEXT          NULL,
    CHECK (ti_le_thu_lao BETWEEN 0 AND 100)
) COMMENT ='Bảng lưu thông tin chính sách thù lao';

-- seed data
INSERT INTO nv_chinh_sach_thu_lao
(`loai_nghiep_vu`, `loai_san_pham`, `phuong_thuc_dong`, `vung_dvhc`, `phuong_an_dong`, `trang_thai`, `ti_le_thu_lao`, `ghi_chu`)
VALUES
    ('BHXH_TU_NGUYEN', 'bhxh', '12', 'I', 'TM', 1, 18.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '6', 'I', 'TM', 1, 16.20, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '3', 'I', 'TM', 1, 13.50, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '1', 'I', 'TM', 1, 10.80, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', 'VS', 'I', 'TM', 1, 18.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '12', 'I', 'TT', 1, 7.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '6', 'I', 'TT', 1, 7.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '3', 'I', 'TT', 1, 7.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '1', 'I', 'TT', 1, 7.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', 'VS', 'I', 'TT', 1, 7.00, NULL),

    ('BHYT_HO_GIA_DINH', 'GD', '12', 'I', 'TM', 1, 8.80, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '6', 'I', 'TM', 1, 7.92, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '3', 'I', 'TM', 1, 6.60, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '12', 'I', 'TT', 1, 3.77, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '6', 'I', 'TT', 1, 3.77, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '3', 'I', 'TT', 1, 3.77, NULL),
    ('BHYT_HO_GIA_DINH', 'GB', '-1', 'I', 'TM', 1, 11.80, NULL),
    ('BHYT_HO_GIA_DINH', 'CN', '-1', 'I', 'TM', 1, 13.80, NULL),
    ('BHYT_HO_GIA_DINH', 'GB', '-1', 'I', 'TT', 1, 11.80, NULL),
    ('BHYT_HO_GIA_DINH', 'CN', '-1', 'I', 'TT', 1, 13.80, NULL),

    ('BHXH_TU_NGUYEN', 'bhxh', '12', 'II', 'TM', 1, 19.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '6', 'II', 'TM', 1, 17.10, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '3', 'II', 'TM', 1, 14.30, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '1', 'II', 'TM', 1, 11.40, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', 'VS', 'II', 'TM', 1, 19.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '12', 'II', 'TT', 1, 8.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '6', 'II', 'TT', 1, 8.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '3', 'II', 'TT', 1, 8.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '1', 'II', 'TT', 1, 8.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', 'VS', 'II', 'TT', 1, 8.00, NULL),

    ('BHYT_HO_GIA_DINH', 'GD', '12', 'II', 'TM', 1, 9.80, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '6', 'II', 'TM', 1, 8.82, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '3', 'II', 'TM', 1, 7.35, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '12', 'II', 'TT', 1, 4.20, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '6', 'II', 'TT', 1, 4.20, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '3', 'II', 'TT', 1, 4.20, NULL),
    ('BHYT_HO_GIA_DINH', 'GB', '-1', 'II', 'TM', 1, 12.80, NULL),
    ('BHYT_HO_GIA_DINH', 'CN', '-1', 'II', 'TM', 1, 14.80, NULL),
    ('BHYT_HO_GIA_DINH', 'GB', '-1', 'II', 'TT', 1, 12.80, NULL),
    ('BHYT_HO_GIA_DINH', 'CN', '-1', 'II', 'TT', 1, 14.80, NULL),

    ('BHXH_TU_NGUYEN', 'bhxh', '12', 'III', 'TM', 1, 20.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '6', 'III', 'TM', 1, 18.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '3', 'III', 'TM', 1, 15.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '1', 'III', 'TM', 1, 13.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', 'VS', 'III', 'TM', 1, 20.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '12', 'III', 'TT', 1, 9.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '6', 'III', 'TT', 1, 9.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '3', 'III', 'TT', 1, 9.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', '1', 'III', 'TT', 1, 9.00, NULL),
    ('BHXH_TU_NGUYEN', 'bhxh', 'VS', 'III', 'TT', 1, 9.00, NULL),

    ('BHYT_HO_GIA_DINH', 'GD', '12', 'III', 'TM', 1, 10.80, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '6', 'III', 'TM', 1, 9.72, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '3', 'III', 'TM', 1, 8.10, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '12', 'III', 'TT', 1, 4.63, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '6', 'III', 'TT', 1, 4.63, NULL),
    ('BHYT_HO_GIA_DINH', 'GD', '3', 'III', 'TT', 1, 4.63, NULL),
    ('BHYT_HO_GIA_DINH', 'GB', '-1', 'III', 'TM', 1, 13.80, NULL),
    ('BHYT_HO_GIA_DINH', 'CN', '-1', 'III', 'TM', 1, 15.80, NULL),
    ('BHYT_HO_GIA_DINH', 'GB', '-1', 'III', 'TT', 1, 13.80, NULL),
    ('BHYT_HO_GIA_DINH', 'CN', '-1', 'III', 'TT', 1, 15.80, NULL);

-- them cột tỷ lệ thù lao - để handle nghiệp vụ tính thù lao theo ngày => log chính xác sẽ được lưu ở các bảng đối soát
ALTER TABLE nv_ho_so_dang_ky
    ADD COLUMN ti_le_thu_lao DECIMAL(5, 2) NOT NULL COMMENT 'Tỷ lệ thù lao % (from nv_chinh_sach_thu_lao)';

CREATE TABLE nv_doi_soat_thu_lao
(
    -- Primary Key
    id                  INT AUTO_INCREMENT PRIMARY KEY,

    -- Business Identifiers
    ten_doi_soat        VARCHAR(200) NOT NULL COMMENT 'Tên đối soát (unique)',

    -- Date Range
    tu_ngay             DATE         NOT NULL COMMENT 'Từ ngày (giai đoạn đối soát)',
    den_ngay            DATE         NOT NULL COMMENT 'Đến ngày (giai đoạn đối soát)',

    -- Foreign Keys
    dai_ly_thu_id       INT          NOT NULL COMMENT 'FK to đại lý thu hộ',
    co_quan_bhxh_id     INT          NOT NULL COMMENT 'FK to cơ quan BHXH chi trả thù lao',

    -- Status
    trang_thai          VARCHAR(20)  NOT NULL DEFAULT 'DANG_DOI_SOAT' COMMENT 'DANG_DOI_SOAT | DA_DOI_SOAT',

    -- Finalization Audit
    nguoi_chot_doi_soat INT NULL COMMENT 'User who finalized (Chốt đối soát)',
    ngay_chot_doi_soat  DATETIME NULL COMMENT 'Timestamp of finalization',

    -- Soft Delete
    deleted_at          DATETIME NULL COMMENT 'Soft delete timestamp',

    -- Audit Timestamps
    created_at          DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by_user_id  INT          NOT NULL COMMENT 'User who created this reconciliation',

    -- Constraints
    CONSTRAINT chk_doi_soat_thu_lao_date_range
        CHECK (den_ngay >= tu_ngay),
    CONSTRAINT chk_doi_soat_thu_lao_trang_thai
        CHECK (trang_thai IN ('DANG_DOI_SOAT', 'DA_DOI_SOAT')),

    -- Unique Constraints
    UNIQUE KEY uk_ten_doi_soat (ten_doi_soat, deleted_at)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
COMMENT='Bảng quản lý kỳ đối soát thù lao';

CREATE TABLE nv_doi_soat_thu_lao_ho_so
(
    -- Primary Key
    id                  INT AUTO_INCREMENT PRIMARY KEY,

    -- Foreign Keys
    doi_soat_thu_lao_id INT            NOT NULL COMMENT 'FK to nv_doi_soat_thu_lao',
    ho_so_id            INT            NOT NULL COMMENT 'FK to nv_ho_so_dang_ky',
    dot_ke_khai_id      INT            NOT NULL COMMENT 'FK to nv_dot_ke_khai',

    -- Denormalized Fields (snapshot at reconciliation time)
    so_tien_dong        DECIMAL(15, 2) NOT NULL COMMENT 'Số tiền đóng (from profile)',
    ti_le_thu_lao       DECIMAL(5, 2)  NOT NULL COMMENT 'Tỷ lệ thù lao % (from nv_chinh_sach_thu_lao)',
    thu_lao_duoc_huong  DECIMAL(15, 2) NOT NULL COMMENT 'Thù lao = so_tien_dong × ti_le_thu_lao',

    -- Additional Profile Metadata (for C17 export)
    ma_ho_so            VARCHAR(50)    NOT NULL COMMENT 'Mã hồ sơ (denormalized)',
    ten_nguoi_tham_gia  VARCHAR(200)   NOT NULL COMMENT 'Tên người tham gia (denormalized)',
    loai_nghiep_vu      VARCHAR(50)    NOT NULL COMMENT 'BHXH_TU_NGUYEN | BHYT_HO_GIA_DINH',
    loai_san_pham       VARCHAR(50)    NOT NULL COMMENT 'BHXH | ma loai doi tuong bao hiem y te dm_loai_doi_tuong_bhyt',
    vung                VARCHAR(10)    NOT NULL COMMENT 'I | II | III (from dm_donvi_hanhchinh)',
    phuong_an_dong      VARCHAR(50) NULL COMMENT 'Phương án đóng',
    phuong_thuc_dong    VARCHAR(50) NULL COMMENT 'Phương thức đóng',
    so_bien_lai         VARCHAR(50) NULL COMMENT 'Số biên lai',

    -- Audit
    ngay_tao            DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    CONSTRAINT chk_doi_soat_thu_lao_ho_so_ti_le
        CHECK (ti_le_thu_lao BETWEEN 0 AND 100),
    CONSTRAINT chk_doi_soat_thu_lao_ho_so_tien
        CHECK (so_tien_dong >= 0 AND thu_lao_duoc_huong >= 0),

    -- Unique: 1 profile → 1 reconciliation period (business rule)
    UNIQUE KEY uk_ho_so_reconciliation (ho_so_id),

    -- Indexes
    INDEX               idx_doi_soat_thu_lao (doi_soat_thu_lao_id)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
COMMENT='Bảng mapping kỳ đối soát thù lao và hồ sơ, lưu tính toán thù lao';


CREATE TABLE nv_doi_soat_hang_ngay
(
    -- Primary Key
    id                INT AUTO_INCREMENT PRIMARY KEY,

    -- Business Identifiers
    so_van_ban        VARCHAR(100) NOT NULL COMMENT 'Số văn bản (unique)',
    ngay_doi_soat     DATE         NOT NULL COMMENT 'Ngày đối soát',

    -- Foreign Keys
    dai_ly_thu_id     INT          NOT NULL COMMENT 'FK to đại lý thu hộ',
    co_quan_bhxh_id   INT          NOT NULL COMMENT 'FK to cơ quan BHXH chi trả thù lao',

    -- Payment Method Details
    hinh_thuc_nop     VARCHAR(50)  NOT NULL COMMENT 'Nộp trực tiếp tại CQ BHXH | Chuyển khoản qua ngân hàng',

    so_giao_dich      VARCHAR(100) NULL
                            COMMENT 'Số giao dịch ngân hàng (if Chuyển khoản) hoặc Số phiếu thu (if Nộp trực tiếp)',

    ngay_giao_dich    DATE NULL COMMENT 'Ngày giao dịch',

    ngan_hang_co_quan VARCHAR(200) NULL
                            COMMENT 'Tên ngân hàng (if Chuyển khoản) hoặc Cơ quan lập phiếu thu (if Nộp trực tiếp)',

    -- Soft Delete
    deleted_at        DATETIME NULL COMMENT 'Soft delete timestamp',

    -- Audit Timestamps
    ngay_tao          DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    ngay_cap_nhat     DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    nguoi_tao         INT          NOT NULL COMMENT 'User who created this C66a',

    -- Unique Constraints
    UNIQUE KEY uk_so_van_ban (so_van_ban, deleted_at),

    -- Indexes
    INDEX             idx_dai_ly_thu (dai_ly_thu_id, deleted_at),
    INDEX             idx_ngay_doi_soat (ngay_doi_soat DESC, deleted_at)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
COMMENT='Bảng quản lý văn bản đề nghị chi trả thù lao hàng ngày (C66a)';


CREATE TABLE nv_doi_soat_hang_ngay_dot_ke_khai
(
    -- Primary Key
    id                    INT AUTO_INCREMENT PRIMARY KEY,

    -- Foreign Keys
    doi_soat_hang_ngay_id INT            NOT NULL COMMENT 'FK to nv_doi_soat_hang_ngay',
    dot_ke_khai_id        INT            NOT NULL COMMENT 'FK to dot_ke_khai (declaration batch)',

    -- Aggregated Fields (calculated from batch's profiles)
    so_luong_ho_so        INT            NOT NULL COMMENT 'Number of profiles in this batch',
    tong_tien             DECIMAL(15, 2) NOT NULL COMMENT 'Total payment amount (sum from profiles)',
    thu_lao_duoc_huong    DECIMAL(15, 2) NOT NULL COMMENT 'Total fee amount (calculated)',

    -- Denormalized Batch Metadata
    ten_dot_ke_khai       VARCHAR(200)   NOT NULL COMMENT 'Tên đợt kê khai (denormalized)',
    loai_thu_tuc          VARCHAR(50)    NOT NULL COMMENT '602 | 603',

    -- Audit
    ngay_tao              DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    CONSTRAINT chk_doi_soat_hang_ngay_so_luong
        CHECK (so_luong_ho_so > 0),
    CONSTRAINT chk_doi_soat_hang_ngay_tien
        CHECK (tong_tien >= 0 AND thu_lao_duoc_huong >= 0),

    -- Unique: 1 batch → 1 C66a document (business rule)
    UNIQUE KEY uk_dot_ke_khai_reconciliation (dot_ke_khai_id)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
COMMENT='Bảng mapping C66a và đợt kê khai, lưu tổng hợp thù lao';

