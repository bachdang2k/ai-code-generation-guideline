# Task 00: Data Model Foundation — Đối Soát Thù Lao

**PAKT**: §5 (Data Model), §6 (Business Logic — Error Codes + State Machine), §9 (DDL đã có)  
**Hours**: 8  
**Dependencies**: None  
**Output**: Entities, Enums, Repositories, State Machine, Error Codes, Permission

> ⚠️ **DB Schema đã có sẵn trong V5** — không tạo migration mới  
> ⚠️ `ChinhSachThuLaoRepository` **KHÔNG cần** — `ti_le_thu_lao` đọc từ `nv_ho_so_dang_ky`

---

## 1. Entities (PAKT §5)

**Package**: `vn.vivas.bh.entity.nghiepvu`

### 1.1 NvDoiSoatThuLao

Maps table `nv_doi_soat_thu_lao` (V5).

```java
package vn.vivas.bh.entity.nghiepvu;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.SQLDelete;
import org.hibernate.annotations.Where;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;
import vn.vivas.bh.enums.TrangThaiDoiSoat;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "nv_doi_soat_thu_lao")
@EntityListeners(AuditingEntityListener.class)
@SQLDelete(sql = "UPDATE nv_doi_soat_thu_lao SET deleted_at = NOW() WHERE id = ?")
@Where(clause = "deleted_at IS NULL")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class NvDoiSoatThuLao {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(name = "ten_doi_soat", nullable = false, length = 200)
    private String tenDoiSoat;

    @Column(name = "tu_ngay", nullable = false)
    private LocalDate tuNgay;

    @Column(name = "den_ngay", nullable = false)
    private LocalDate denNgay;

    @Column(name = "dai_ly_thu_id", nullable = false)
    private Integer daiLyThuId;

    @Column(name = "co_quan_bhxh_id", nullable = false)
    private Integer coQuanBhxhId;

    @Enumerated(EnumType.STRING)
    @Column(name = "trang_thai", nullable = false, length = 20)
    private TrangThaiDoiSoat trangThai;

    @Column(name = "nguoi_chot_doi_soat")
    private Integer nguoiChotDoiSoat;

    @Column(name = "ngay_chot_doi_soat")
    private LocalDateTime ngayChotDoiSoat;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;

    @Column(name = "created_by_user_id", nullable = false)
    private Integer createdByUserId;
}
```

### 1.2 NvDoiSoatThuLaoHoSo

Maps table `nv_doi_soat_thu_lao_ho_so` (V5). Snapshot data — không thay đổi sau khi tạo.

```java
package vn.vivas.bh.entity.nghiepvu;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "nv_doi_soat_thu_lao_ho_so")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class NvDoiSoatThuLaoHoSo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(name = "doi_soat_thu_lao_id", nullable = false)
    private Integer doiSoatThuLaoId;

    @Column(name = "ho_so_id", nullable = false)
    private Integer hoSoId;

    @Column(name = "dot_ke_khai_id", nullable = false)
    private Integer dotKeKhaiId;

    // ─── Snapshot financial fields ───────────────────────────────────────────
    @Column(name = "so_tien_dong", nullable = false, precision = 15, scale = 2)
    private BigDecimal soTienDong;

    @Column(name = "ti_le_thu_lao", nullable = false, precision = 5, scale = 2)
    private BigDecimal tiLeThuLao;

    @Column(name = "thu_lao_duoc_huong", nullable = false, precision = 15, scale = 2)
    private BigDecimal thuLaoDuocHuong;

    // ─── Denormalized metadata (snapshot at reconciliation time) ─────────────
    @Column(name = "ma_ho_so", nullable = false, length = 50)
    private String maHoSo;

    @Column(name = "ten_nguoi_tham_gia", nullable = false, length = 200)
    private String tenNguoiThamGia;

    @Column(name = "loai_nghiep_vu", nullable = false, length = 50)
    private String loaiNghiepVu;

    @Column(name = "loai_san_pham", nullable = false, length = 50)
    private String loaiSanPham;

    @Column(name = "vung", nullable = false, length = 10)
    private String vung;

    @Column(name = "phuong_an_dong", length = 50)
    private String phuongAnDong;

    @Column(name = "phuong_thuc_dong", length = 50)
    private String phuongThucDong;

    @Column(name = "so_bien_lai", length = 50)
    private String soBienLai;

    @Column(name = "ngay_tao", nullable = false)
    private LocalDateTime ngayTao;
}
```

---

## 2. Enums

**Package**: `vn.vivas.bh.enums`

```java
package vn.vivas.bh.enums;

import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public enum TrangThaiDoiSoat {

    DANG_DOI_SOAT("Đang đối soát"),
    DA_DOI_SOAT("Đã đối soát");

    private final String ten;
}
```

---

## 3. State Machine

**Package**: `vn.vivas.bh.statemachine`

```java
package vn.vivas.bh.statemachine;

import vn.vivas.bh.constant.MessageResponseDict;
import vn.vivas.bh.enums.TrangThaiDoiSoat;
import vn.vivas.bh.exception.BusinessException;

public class DoiSoatThuLaoStateMachine {

    /**
     * Validate rằng kỳ đối soát có thể bị modify (chốt / sửa / xóa).
     * DA_DOI_SOAT là trạng thái cuối — không cho phép bất kỳ thao tác nào.
     */
    public static void validateCoTheThaoTac(TrangThaiDoiSoat trangThai) {
        if (TrangThaiDoiSoat.DA_DOI_SOAT.equals(trangThai)) {
            throw new BusinessException(MessageResponseDict.DOI_SOAT_INVALID_STATE);
        }
    }

    /**
     * Transition DANG_DOI_SOAT → DA_DOI_SOAT (Chốt kỳ đối soát).
     */
    public static TrangThaiDoiSoat chotDoiSoat(TrangThaiDoiSoat trangThai) {
        validateCoTheThaoTac(trangThai);
        return TrangThaiDoiSoat.DA_DOI_SOAT;
    }
}
```

---

## 4. Repositories (PAKT §4)

**Package**: `vn.vivas.bh.repository.nghiepvu`

### 4.1 DoiSoatThuLaoRepository

```java
package vn.vivas.bh.repository.nghiepvu;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import vn.vivas.bh.entity.nghiepvu.NvDoiSoatThuLao;
import vn.vivas.bh.repository.projection.DoiSoatListProjection;

@Repository
public interface DoiSoatThuLaoRepository extends JpaRepository<NvDoiSoatThuLao, Integer> {

    boolean existsByTenDoiSoat(String tenDoiSoat);

    /**
     * Danh sách kỳ đối soát với aggregation COUNT/SUM từ bảng con,
     * JOIN agency + dm_co_quan_bhxh để lấy tên display.
     * RBAC filter:
     *   - Admin: :createdByUserId NULL → hiển thị tất cả (hoặc filter theo daiLyThuId)
     *   - Đại lý thu: :createdByUserId NOT NULL → chỉ own records
     *
     * DB: nv_doi_soat_thu_lao (ds)
     *     LEFT JOIN agency (a) ON a.id = ds.dai_ly_thu_id
     *     LEFT JOIN dm_co_quan_bhxh (cq) ON cq.id = ds.co_quan_bhxh_id
     *     LEFT JOIN subquery agg (COUNT/SUM từ nv_doi_soat_thu_lao_ho_so)
     */
    @Query(value = """
            SELECT ds.id                      AS id,
                   ds.ten_doi_soat            AS tenDoiSoat,
                   ds.tu_ngay                 AS tuNgay,
                   ds.den_ngay                AS denNgay,
                   ds.trang_thai              AS trangThai,
                   a.name                     AS tenDaiLyThu,
                   cq.ten                     AS tenCoQuanBhxh,
                   COALESCE(agg.soLuongHoSo, 0) AS soLuongHoSo,
                   COALESCE(agg.tongTien, 0)    AS tongTien,
                   COALESCE(agg.tongThuLao, 0)  AS tongThuLao
            FROM nv_doi_soat_thu_lao ds
            LEFT JOIN agency a ON a.id = ds.dai_ly_thu_id
            LEFT JOIN dm_co_quan_bhxh cq ON cq.id = ds.co_quan_bhxh_id
            LEFT JOIN (
                SELECT doi_soat_thu_lao_id,
                       COUNT(*)                 AS soLuongHoSo,
                       SUM(so_tien_dong)        AS tongTien,
                       SUM(thu_lao_duoc_huong)  AS tongThuLao
                FROM nv_doi_soat_thu_lao_ho_so
                GROUP BY doi_soat_thu_lao_id
            ) agg ON agg.doi_soat_thu_lao_id = ds.id
            WHERE ds.deleted_at IS NULL
              AND (:daiLyThuId IS NULL OR ds.dai_ly_thu_id = :daiLyThuId)
              AND (:trangThai IS NULL OR ds.trang_thai = :trangThai)
              AND (:tenSearch IS NULL OR ds.ten_doi_soat LIKE CONCAT('%', :tenSearch, '%'))
              AND (:tuNgay IS NULL OR ds.tu_ngay >= :tuNgay)
              AND (:denNgay IS NULL OR ds.den_ngay <= :denNgay)
              AND (:createdByUserId IS NULL OR ds.created_by_user_id = :createdByUserId)
            ORDER BY ds.created_at DESC
            """,
            countQuery = """
            SELECT COUNT(*) FROM nv_doi_soat_thu_lao ds
            WHERE ds.deleted_at IS NULL
              AND (:daiLyThuId IS NULL OR ds.dai_ly_thu_id = :daiLyThuId)
              AND (:trangThai IS NULL OR ds.trang_thai = :trangThai)
              AND (:tenSearch IS NULL OR ds.ten_doi_soat LIKE CONCAT('%', :tenSearch, '%'))
              AND (:tuNgay IS NULL OR ds.tu_ngay >= :tuNgay)
              AND (:denNgay IS NULL OR ds.den_ngay <= :denNgay)
              AND (:createdByUserId IS NULL OR ds.created_by_user_id = :createdByUserId)
            """,
            nativeQuery = true)
    Page<DoiSoatListProjection> findAllWithFilter(
            @Param("daiLyThuId") Integer daiLyThuId,
            @Param("trangThai") String trangThai,
            @Param("tenSearch") String tenSearch,
            @Param("tuNgay") String tuNgay,
            @Param("denNgay") String denNgay,
            @Param("createdByUserId") Integer createdByUserId,
            Pageable pageable
    );

    /**
     * Đợt kê khai hợp lệ để tạo kỳ đối soát:
     *   - Thuộc đại lý thu (qua dai_ly_id)
     *   - Status IN (CQ_BHXH_DANG_XU_LY, THANH_CONG)
     *   - Trong giai đoạn OR tồn đọ ng trước tuNgay với hồ sơ chưa đối soát
     *   - COUNT hồ sơ chưa đối soát > 0
     *
     * Ghi chú:
     *   - `nv_dot_ke_khai` không có cột ten_dot_ke_khai — được tính trong Java:
     *     String.format("[%02d] %s", soThuTu, createdAt.format(MM/yyyy))
     *   - Hồ sơ link đến đợt qua bảng nối nv_dot_ke_khai_ho_so
     *   - Column dai_ly_id (không phải dai_ly_thu_id)
     *   - Column thoi_gian_gui (không phải ngay_gui)
     *
     * DB: nv_dot_ke_khai (dkk)
     *     JOIN nv_dot_ke_khai_ho_so (dkk_hs) ON dkk_hs.dot_ke_khai_id = dkk.id
     *     LEFT JOIN nv_doi_soat_thu_lao_ho_so (dshs) ON dshs.ho_so_id = dkk_hs.ho_so_id
     *
     * Service phải xử lý: tenDotKeKhai = String.format("[%02d] %s",
     *     p.getSoThuTu(), p.getCreatedAt().format(DateTimeFormatter.ofPattern("MM/yyyy")))
     */
    @Query(value = """
            SELECT dkk.id                        AS dotKeKhaiId,
                   dkk.so_thu_tu                 AS soThuTu,
                   dkk.created_at                AS createdAt,
                   dkk.loai_thu_tuc              AS thuTuc,
                   COUNT(CASE WHEN dshs.ho_so_id IS NULL THEN dkk_hs.ho_so_id END) AS soLuongHoSoChuaDoiSoat
            FROM nv_dot_ke_khai dkk
            JOIN nv_dot_ke_khai_ho_so dkk_hs ON dkk_hs.dot_ke_khai_id = dkk.id
            LEFT JOIN nv_doi_soat_thu_lao_ho_so dshs ON dshs.ho_so_id = dkk_hs.ho_so_id
            WHERE dkk.dai_ly_id = :daiLyThuId
              AND dkk.trang_thai IN ('CQ_BHXH_DANG_XU_LY', 'THANH_CONG')
              AND (
                  (dkk.thoi_gian_gui BETWEEN :tuNgay AND :denNgay)
                  OR (dkk.thoi_gian_gui < :tuNgay)
              )
            GROUP BY dkk.id, dkk.so_thu_tu, dkk.created_at, dkk.loai_thu_tuc
            HAVING soLuongHoSoChuaDoiSoat > 0
            ORDER BY dkk.thoi_gian_gui ASC
            """,
            nativeQuery = true)
    java.util.List<vn.vivas.bh.repository.projection.DotKeKhaiHopLeProjection> findDotKeKhaiHopLe(
            @Param("daiLyThuId") Integer daiLyThuId,
            @Param("tuNgay") String tuNgay,
            @Param("denNgay") String denNgay
    );
}
```

### 4.2 DoiSoatThuLaoHoSoRepository

```java
package vn.vivas.bh.repository.nghiepvu;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import vn.vivas.bh.entity.nghiepvu.NvDoiSoatThuLaoHoSo;
import vn.vivas.bh.repository.projection.DoiSoatHoSoDetailProjection;

import java.util.List;

@Repository
public interface DoiSoatThuLaoHoSoRepository extends JpaRepository<NvDoiSoatThuLaoHoSo, Integer> {

    /**
     * Xóa các hồ sơ khỏi kỳ đối soát theo danh sách hoSoId.
     * Dùng khi xóa toàn bộ (xóa kỳ) hoặc xóa lẻ (sửa kỳ).
     * BATCH DELETE — không loop từng id.
     */
    @Modifying
    @Query("DELETE FROM NvDoiSoatThuLaoHoSo h WHERE h.doiSoatThuLaoId = :doiSoatId AND h.hoSoId IN :hoSoIds")
    int deleteByDoiSoatThuLaoIdAndHoSoIdIn(
            @Param("doiSoatId") Integer doiSoatId,
            @Param("hoSoIds") List<Integer> hoSoIds
    );

    /**
     * Xóa toàn bộ hồ sơ của 1 kỳ đối soát (dùng khi xóa kỳ).
     */
    @Modifying
    @Query("DELETE FROM NvDoiSoatThuLaoHoSo h WHERE h.doiSoatThuLaoId = :doiSoatId")
    int deleteAllByDoiSoatThuLaoId(@Param("doiSoatId") Integer doiSoatId);

    /**
     * Đếm số hồ sơ đã thuộc kỳ đối soát nào đó (từ danh sách hoSoIds).
     * Dùng để validate khi thêm hồ sơ lẻ.
     */
    @Query("SELECT h.hoSoId FROM NvDoiSoatThuLaoHoSo h WHERE h.hoSoId IN :hoSoIds")
    List<Integer> findHoSoIdsDaDoiSoat(@Param("hoSoIds") List<Integer> hoSoIds);

    /**
     * Chi tiết hồ sơ trong kỳ đối soát — phân trang.
     * Đọc từ snapshot nv_doi_soat_thu_lao_ho_so (không JOIN thêm).
     *
     * DB: nv_doi_soat_thu_lao_ho_so (dshs)
     *     [dm fields phuongAnDong/phuongThucDong được resolve bằng Java key-value map ở service]
     */
    Page<NvDoiSoatThuLaoHoSo> findAllByDoiSoatThuLaoId(Integer doiSoatThuLaoId, Pageable pageable);

    /**
     * Load toàn bộ hồ sơ của 1 kỳ (cho export XLSX — không phân trang).
     */
    List<NvDoiSoatThuLaoHoSo> findAllByDoiSoatThuLaoId(Integer doiSoatThuLaoId);

    /**
     * Hồ sơ lᮧ có thể thêm vào kỳ đang sửa:
     *   - Trạng thái THANH_CONG hoặc CQ_BHXH_DANG_XU_LY
     *   - Chưa thuộc kỳ đối soát nào (ho_so_id NOT IN nv_doi_soat_thu_lao_ho_so)
     *   - Thuộc đại lý thu (qua nv_dot_ke_khai_ho_so JOIN nv_dot_ke_khai.dai_ly_id)
     *
     * Ghi chú: `tenDotKeKhai` không có trong DB — được tính trong Java service từ
     *   so_thu_tu + created_at: String.format("[%02d] %s", soThuTu, createdAt.format(MM/yyyy))
     *
     * DB: nv_ho_so_dang_ky (hs)
     *     LEFT JOIN nv_nguoi_tham_gia (ntt) ON ntt.ho_so_id = hs.id
     *     JOIN nv_dot_ke_khai_ho_so (dkk_hs) ON dkk_hs.ho_so_id = hs.id
     *     JOIN nv_dot_ke_khai (dkk) ON dkk.id = dkk_hs.dot_ke_khai_id
     */
    @Query(value = """
            SELECT hs.id                  AS hoSoId,
                   hs.ma_ho_so            AS maHoSo,
                   dkk.so_thu_tu          AS soThuTu,
                   dkk.created_at         AS dotKeKhaiCreatedAt,
                   ntt.ho_ten             AS tenNguoiThamGia,
                   hs.loai_nghiep_vu      AS thuTuc,
                   hs.trang_thai          AS trangThai
            FROM nv_ho_so_dang_ky hs
            LEFT JOIN nv_nguoi_tham_gia ntt ON ntt.ho_so_id = hs.id
            JOIN nv_dot_ke_khai_ho_so dkk_hs ON dkk_hs.ho_so_id = hs.id
            JOIN nv_dot_ke_khai dkk ON dkk.id = dkk_hs.dot_ke_khai_id
            WHERE hs.trang_thai IN ('THANH_CONG', 'CQ_BHXH_DANG_XU_LY')
              AND dkk.dai_ly_id = :daiLyThuId
              AND hs.id NOT IN (SELECT ho_so_id FROM nv_doi_soat_thu_lao_ho_so)
              AND (:keyword IS NULL
                  OR hs.ma_ho_so LIKE CONCAT('%', :keyword, '%')
                  OR ntt.ho_ten LIKE CONCAT('%', :keyword, '%'))
            ORDER BY hs.ngay_tao DESC
            """,
            countQuery = """
            SELECT COUNT(*) FROM nv_ho_so_dang_ky hs
            LEFT JOIN nv_nguoi_tham_gia ntt ON ntt.ho_so_id = hs.id
            JOIN nv_dot_ke_khai_ho_so dkk_hs ON dkk_hs.ho_so_id = hs.id
            JOIN nv_dot_ke_khai dkk ON dkk.id = dkk_hs.dot_ke_khai_id
            WHERE hs.trang_thai IN ('THANH_CONG', 'CQ_BHXH_DANG_XU_LY')
              AND dkk.dai_ly_id = :daiLyThuId
              AND hs.id NOT IN (SELECT ho_so_id FROM nv_doi_soat_thu_lao_ho_so)
              AND (:keyword IS NULL
                  OR hs.ma_ho_so LIKE CONCAT('%', :keyword, '%')
                  OR ntt.ho_ten LIKE CONCAT('%', :keyword, '%'))
            """,
            nativeQuery = true)
    Page<vn.vivas.bh.repository.projection.HoSoLeProjection> findHoSoLe(
            @Param("daiLyThuId") Integer daiLyThuId,
            @Param("keyword") String keyword,
            Pageable pageable
    );
}
```

---

## 5. Projections

**Package**: `vn.vivas.bh.repository.projection`

```java
package vn.vivas.bh.repository.projection;

import java.math.BigDecimal;
import java.time.LocalDate;

/** Cho GET danh sách kỳ đối soát */
public interface DoiSoatListProjection {
    Integer getId();
    String getTenDoiSoat();
    LocalDate getTuNgay();
    LocalDate getDenNgay();
    String getTrangThai();
    String getTenDaiLyThu();
    String getTenCoQuanBhxh();
    Long getSoLuongHoSo();
    BigDecimal getTongTien();
    BigDecimal getTongThuLao();
}
```

```java
package vn.vivas.bh.repository.projection;

import java.time.LocalDateTime;

/**
 * Cho GET dot-ke-khai-hop-le.
 * tenDotKeKhai không có trong DB — service tính từ soThuTu + createdAt:
 *   String.format("[%02d] %s", p.getSoThuTu(),
 *       p.getCreatedAt().format(DateTimeFormatter.ofPattern("MM/yyyy")))
 */
public interface DotKeKhaiHopLeProjection {
    Integer getDotKeKhaiId();
    Integer getSoThuTu();
    LocalDateTime getCreatedAt();
    String getThuTuc();
    Long getSoLuongHoSoChuaDoiSoat();
}
```

```java
package vn.vivas.bh.repository.projection;

import java.time.LocalDateTime;

/**
 * Cho GET hồ sơ lẻ.
 * tenDotKeKhai không có trong DB — service tính từ soThuTu + dotKeKhaiCreatedAt:
 *   String.format("[%02d] %s", p.getSoThuTu(),
 *       p.getDotKeKhaiCreatedAt().format(DateTimeFormatter.ofPattern("MM/yyyy")))
 */
public interface HoSoLeProjection {
    Integer getHoSoId();
    String getMaHoSo();
    Integer getSoThuTu();
    LocalDateTime getDotKeKhaiCreatedAt();
    String getTenNguoiThamGia();
    String getThuTuc();
    String getTrangThai();
}
```

---

## 6. Error Codes (PAKT §6 — thêm vào `MessageResponseDict`)

**Range**: 3200xx

```java
// Thêm vào enum MessageResponseDict:

// ─── Đối Soát Thù Lao (3200xx) ────────────────────────────────────────────────
DOI_SOAT_NOT_FOUND(HttpStatus.NOT_FOUND, 320001, "Kỳ đối soát không tồn tại"),
DOI_SOAT_TEN_DUPLICATE(HttpStatus.OK, 320002, "Tên đối soát đã tồn tại"),
INVALID_DATE_RANGE(HttpStatus.OK, 320003, "Giai đoạn đối soát không hợp lệ"),
DOI_SOAT_EMPTY_DOT(HttpStatus.OK, 320004, "Vui lòng chọn tối thiểu 1 đợt kê khai"),
DOI_SOAT_INVALID_STATE(HttpStatus.OK, 320005, "Kỳ đối soát đã chốt, không thể thực hiện thao tác này"),
DOI_SOAT_FORBIDDEN(HttpStatus.FORBIDDEN, 320006, "Bạn không có quyền thực hiện thao tác này"),
```

---

## 7. Permission (thêm vào `PermissionConstants`)

```java
// Thêm vào class PermissionConstants:

public static final String DOI_SOAT_THU_LAO = "DOI_SOAT_THU_LAO";
```

---

## 8. Repository Integration Test

**Class**: `DoiSoatThuLaoRepositoryIT`  
**Package**: `vn.vivas.bh.repository`

```java
package vn.vivas.bh.repository;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.context.annotation.Import;
import org.springframework.data.domain.PageRequest;
import vn.vivas.bh.config.TestcontainersConfiguration;
import vn.vivas.bh.entity.nghiepvu.NvDoiSoatThuLao;
import vn.vivas.bh.enums.TrangThaiDoiSoat;
import vn.vivas.bh.repository.nghiepvu.DoiSoatThuLaoRepository;

import java.time.LocalDate;
import java.time.LocalDateTime;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@Import(TestcontainersConfiguration.class)
class DoiSoatThuLaoRepositoryIT {

    @Autowired
    private DoiSoatThuLaoRepository repository;

    @BeforeEach
    void setUp() {
        repository.deleteAll();
    }

    @Test
    void shouldSaveAndFindById() {
        // Given
        NvDoiSoatThuLao doiSoat = NvDoiSoatThuLao.builder()
                .tenDoiSoat("Kỳ đối soát tháng 01/2026")
                .tuNgay(LocalDate.of(2026, 1, 1))
                .denNgay(LocalDate.of(2026, 1, 31))
                .daiLyThuId(1)
                .coQuanBhxhId(1)
                .trangThai(TrangThaiDoiSoat.DANG_DOI_SOAT)
                .createdByUserId(1)
                .build();

        // When
        NvDoiSoatThuLao saved = repository.save(doiSoat);

        // Then
        assertThat(repository.findById(saved.getId())).isPresent();
    }

    @Test
    void shouldReturnTrueWhenTenDoiSoatExists() {
        // Given
        NvDoiSoatThuLao doiSoat = NvDoiSoatThuLao.builder()
                .tenDoiSoat("Kỳ trùng tên")
                .tuNgay(LocalDate.now().minusMonths(1))
                .denNgay(LocalDate.now())
                .daiLyThuId(1)
                .coQuanBhxhId(1)
                .trangThai(TrangThaiDoiSoat.DANG_DOI_SOAT)
                .createdByUserId(1)
                .build();
        repository.save(doiSoat);

        // Then
        assertThat(repository.existsByTenDoiSoat("Kỳ trùng tên")).isTrue();
        assertThat(repository.existsByTenDoiSoat("Tên khác")).isFalse();
    }

    @Test
    void shouldSoftDeleteAndNotReturnInQuery() {
        // Given
        NvDoiSoatThuLao doiSoat = NvDoiSoatThuLao.builder()
                .tenDoiSoat("Kỳ sẽ bị xóa")
                .tuNgay(LocalDate.now().minusMonths(1))
                .denNgay(LocalDate.now())
                .daiLyThuId(1)
                .coQuanBhxhId(1)
                .trangThai(TrangThaiDoiSoat.DANG_DOI_SOAT)
                .createdByUserId(1)
                .build();
        NvDoiSoatThuLao saved = repository.save(doiSoat);

        // When: soft delete
        repository.delete(saved);

        // Then: @Where(clause = "deleted_at IS NULL") hides it
        assertThat(repository.findById(saved.getId())).isEmpty();
    }
}
```

---

## 9. Checklist

- [ ] Entity `NvDoiSoatThuLao` — đầy đủ annotations, @SQLDelete, @Where
- [ ] Entity `NvDoiSoatThuLaoHoSo` — snapshot fields đủ theo V5
- [ ] Enum `TrangThaiDoiSoat` — @Getter @AllArgsConstructor
- [ ] `DoiSoatThuLaoStateMachine` — validateCoTheThaoTac, chotDoiSoat
- [ ] `DoiSoatThuLaoRepository` — findAllWithFilter (native), findDotKeKhaiHopLe (native)
- [ ] `DoiSoatThuLaoHoSoRepository` — deleteAll, deleteByIds, findHoSoLe (native)
- [ ] Projections: DoiSoatListProjection, DotKeKhaiHopLeProjection, HoSoLeProjection
- [ ] Error codes thêm vào `MessageResponseDict` (range 3200xx)
- [ ] Permission `DOI_SOAT_THU_LAO` thêm vào `PermissionConstants`
- [ ] `DoiSoatThuLaoRepositoryIT` — save/find, existsByTen, soft delete
- [ ] `./mvnw clean compile` — no errors

## 10. Sync Points

**Provides (for Task 01, 02, 03)**:
- `NvDoiSoatThuLao`, `NvDoiSoatThuLaoHoSo` entities
- `DoiSoatThuLaoRepository`, `DoiSoatThuLaoHoSoRepository`
- `TrangThaiDoiSoat` enum
- `DoiSoatThuLaoStateMachine`
- `MessageResponseDict` error codes (3200xx)
- `PermissionConstants.DOI_SOAT_THU_LAO`
- Projections: `DoiSoatListProjection`, `DotKeKhaiHopLeProjection`, `HoSoLeProjection`
