# Task 01: List + Create API — Đối Soát Thù Lao

**PAKT**: §2, §4 (API + Query Logic), §6 (Service Methods), §8 (Security)  
**Hours**: 10  
**Dependencies**: Task 00 ✅

---

## 1. API Contract

| Method | Path | Request | Response | Errors |
|--------|------|---------|----------|--------|
| GET | `/api/portal/v1/doi-soat-thu-lao` | Query params | `Page<DoiSoatThuLaoListItemResponse>` | — |
| POST | `/api/portal/v1/doi-soat-thu-lao` | `TaoDoiSoatThuLaoRequest` | `DoiSoatThuLaoDetailResponse` | DOI_SOAT_TEN_DUPLICATE, INVALID_DATE_RANGE, DOI_SOAT_EMPTY_DOT |
| GET | `/api/portal/v1/doi-soat-thu-lao/dot-ke-khai-hop-le` | Query params | `List<DotKeKhaiHopLeResponse>` | — |

---

## 2. Request DTOs

**Package**: `vn.vivas.bh.request.doisoat`

```java
package vn.vivas.bh.request.doisoat;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import lombok.Data;
import java.time.LocalDate;
import java.util.List;

@Data
public class TaoDoiSoatThuLaoRequest {

    @NotBlank(message = "Tên đối soát không được để trống")
    @Size(max = 200)
    private String tenDoiSoat;

    @NotNull(message = "Đại lý thu không được để trống")
    private Integer daiLyThuId;

    @NotNull(message = "Cơ quan BHXH không được để trống")
    private Integer coQuanBhxhId;

    @NotNull
    private LocalDate tuNgay;

    @NotNull
    private LocalDate denNgay;

    @NotNull
    @Size(min = 1, message = "Vui lòng chọn tối thiểu 1 đợt kê khai")
    private List<Integer> dotKeKhaiIds;
}
```

---

## 3. Response DTOs

**Package**: `vn.vivas.bh.response.doisoat`

```java
// DoiSoatThuLaoListItemResponse.java
package vn.vivas.bh.response.doisoat;

import lombok.Builder;
import lombok.Data;
import java.math.BigDecimal;
import java.time.LocalDate;

@Data @Builder
public class DoiSoatThuLaoListItemResponse {
    private Integer id;
    private String tenDoiSoat;
    private LocalDate tuNgay;
    private LocalDate denNgay;
    private String tenDaiLyThu;
    private String tenCoQuanBhxh;
    private Long soLuongHoSo;
    private BigDecimal tongTien;
    private BigDecimal thuLaoDuocHuong;
    private String trangThai;     // enum name
    private String tenTrangThai;  // enum.getTen()
}
```

```java
// DoiSoatThuLaoDetailResponse.java
package vn.vivas.bh.response.doisoat;

import lombok.Builder;
import lombok.Data;
import org.springframework.data.domain.Page;
import java.math.BigDecimal;
import java.time.LocalDate;

@Data @Builder
public class DoiSoatThuLaoDetailResponse {
    private Integer id;
    private String tenDoiSoat;
    private LocalDate tuNgay;
    private LocalDate denNgay;
    private String tenDaiLyThu;
    private String tenCoQuanBhxh;
    private String trangThai;
    private String tenTrangThai;
    private Page<DoiSoatHoSoDetailItem> hoSoList;
    private BigDecimal tongSoTienDong;  // total all pages
    private BigDecimal tongThuLao;      // total all pages
}
```

```java
// DoiSoatHoSoDetailItem.java
package vn.vivas.bh.response.doisoat;

import lombok.Builder;
import lombok.Data;
import java.math.BigDecimal;

@Data @Builder
public class DoiSoatHoSoDetailItem {
    private Integer id;
    private String maHoSo;
    private String tenNguoiThamGia;
    private String thuTuc;          // BHXH Tự nguyện | BHYT Hộ gia đình
    private String vung;            // I | II | III
    private String phuongAnDong;    // mã → tên (resolved by Java map)
    private String phuongThucDong;  // mã → tên (resolved by Java map)
    private String soBienLai;
    private BigDecimal soTienDong;
    private BigDecimal tiLeThuLao;
    private BigDecimal thuLaoDuocHuong;
}
```

```java
// DotKeKhaiHopLeResponse.java
package vn.vivas.bh.response.doisoat;

import lombok.Builder;
import lombok.Data;

@Data @Builder
public class DotKeKhaiHopLeResponse {
    private Integer dotKeKhaiId;
    private String tenDotKeKhai;
    private String thuTuc;                // 602 | 603
    private Long soLuongHoSoChuaDoiSoat;
}
```

---

## 4. Service Methods

**Class**: `DoiSoatThuLaoService` | **Package**: `vn.vivas.bh.service`

```java
package vn.vivas.bh.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import vn.vivas.bh.auth.AccountInfo;
import vn.vivas.bh.auth.AuthenticationService;
import vn.vivas.bh.constant.MessageResponseDict;
import vn.vivas.bh.entity.nghiepvu.NvDoiSoatThuLao;
import vn.vivas.bh.entity.nghiepvu.NvDoiSoatThuLaoHoSo;
import vn.vivas.bh.enums.TrangThaiDoiSoat;
import vn.vivas.bh.exception.BusinessException;
import vn.vivas.bh.repository.nghiepvu.DoiSoatThuLaoHoSoRepository;
import vn.vivas.bh.repository.nghiepvu.DoiSoatThuLaoRepository;
import vn.vivas.bh.repository.nghiepvu.HoSoRepository;
import vn.vivas.bh.repository.projection.HoSoForDoiSoatProjection;
import vn.vivas.bh.request.doisoat.TaoDoiSoatThuLaoRequest;
import vn.vivas.bh.response.doisoat.*;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class DoiSoatThuLaoService {

    private final DoiSoatThuLaoRepository doiSoatRepository;
    private final DoiSoatThuLaoHoSoRepository doiSoatHoSoRepository;
    private final HoSoRepository hoSoRepository;

    // ─── GET Danh sách ─────────────────────────────────────────────────────────
    public Page<DoiSoatThuLaoListItemResponse> getDoiSoatList(
            Integer daiLyThuId, String trangThai, String tenSearch,
            LocalDate tuNgay, LocalDate denNgay, Pageable pageable) {

        AccountInfo user = AuthenticationService.getCurrentUserRequired();

        // RBAC scope
        Integer filterDaiLyThuId = user.isAdmin() ? daiLyThuId : null;
        Integer filterCreatedByUserId = user.isAdmin() ? null : user.getId();

        Page<vn.vivas.bh.repository.projection.DoiSoatListProjection> projections =
                doiSoatRepository.findAllWithFilter(
                        filterDaiLyThuId, trangThai, tenSearch,
                        tuNgay != null ? tuNgay.toString() : null,
                        denNgay != null ? denNgay.toString() : null,
                        filterCreatedByUserId, pageable);

        return projections.map(p -> DoiSoatThuLaoListItemResponse.builder()
                .id(p.getId())
                .tenDoiSoat(p.getTenDoiSoat())
                .tuNgay(p.getTuNgay())
                .denNgay(p.getDenNgay())
                .trangThai(p.getTrangThai())
                .tenTrangThai(TrangThaiDoiSoat.valueOf(p.getTrangThai()).getTen())
                .tenDaiLyThu(p.getTenDaiLyThu())
                .tenCoQuanBhxh(p.getTenCoQuanBhxh())
                .soLuongHoSo(p.getSoLuongHoSo())
                .tongTien(p.getTongTien())
                .thuLaoDuocHuong(p.getTongThuLao())
                .build());
    }

    // ─── POST Tạo kỳ đối soát ──────────────────────────────────────────────────
    @Transactional
    public DoiSoatThuLaoDetailResponse taoDoiSoatThuLao(TaoDoiSoatThuLaoRequest request) {
        // 1. Auth
        AccountInfo user = AuthenticationService.getCurrentUserRequired();
        log.info("[{}/TAO_DOI_SOAT] tenDoiSoat={}", user.getId(), request.getTenDoiSoat());

        // 2. Validate unique name
        if (doiSoatRepository.existsByTenDoiSoat(request.getTenDoiSoat())) {
            throw new BusinessException(MessageResponseDict.DOI_SOAT_TEN_DUPLICATE);
        }

        // 3. Validate date range
        LocalDate today = LocalDate.now();
        if (request.getDenNgay().isAfter(today) || request.getTuNgay().isAfter(today)
                || request.getTuNgay().isAfter(request.getDenNgay())) {
            throw new BusinessException(MessageResponseDict.INVALID_DATE_RANGE);
        }

        // 4. Validate dotKeKhaiIds
        if (request.getDotKeKhaiIds() == null || request.getDotKeKhaiIds().isEmpty()) {
            throw new BusinessException(MessageResponseDict.DOI_SOAT_EMPTY_DOT);
        }

        // 5. Batch load hồ sơ — 1 query
        List<HoSoForDoiSoatProjection> hoSoList =
                hoSoRepository.findAllByDotKeKhaiIdIn(request.getDotKeKhaiIds());
        log.info("[{}/TAO_DOI_SOAT] loaded hoSo count={}", user.getId(), hoSoList.size());

        // 6. Build entity
        NvDoiSoatThuLao doiSoat = NvDoiSoatThuLao.builder()
                .tenDoiSoat(request.getTenDoiSoat())
                .tuNgay(request.getTuNgay())
                .denNgay(request.getDenNgay())
                .daiLyThuId(request.getDaiLyThuId())
                .coQuanBhxhId(request.getCoQuanBhxhId())
                .trangThai(TrangThaiDoiSoat.DANG_DOI_SOAT)
                .createdByUserId(user.getId())
                .build();
        doiSoatRepository.save(doiSoat);

        // 7. Build snapshots
        List<NvDoiSoatThuLaoHoSo> snapshots = new ArrayList<>();
        for (HoSoForDoiSoatProjection hoSo : hoSoList) {
            BigDecimal tiLe = hoSo.getTiLeThuLao() != null ? hoSo.getTiLeThuLao() : BigDecimal.ZERO;
            BigDecimal soTien = hoSo.getSoTienPhaiDong() != null ? hoSo.getSoTienPhaiDong() : BigDecimal.ZERO;
            BigDecimal thuLao = soTien.multiply(tiLe).divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);

            snapshots.add(NvDoiSoatThuLaoHoSo.builder()
                    .doiSoatThuLaoId(doiSoat.getId())
                    .hoSoId(hoSo.getId())
                    .dotKeKhaiId(hoSo.getDotKeKhaiId())
                    .soTienDong(soTien)
                    .tiLeThuLao(tiLe)
                    .thuLaoDuocHuong(thuLao)
                    .maHoSo(hoSo.getMaHoSo())
                    .tenNguoiThamGia(hoSo.getTenNguoiThamGia())
                    .loaiNghiepVu(hoSo.getLoaiNghiepVu())
                    .loaiSanPham(hoSo.getLoaiSanPham())
                    .vung(hoSo.getVung() != null ? hoSo.getVung() : "II")
                    .phuongAnDong(hoSo.getPhuongAnDong())
                    .phuongThucDong(hoSo.getPhuongThucDong())
                    .soBienLai(hoSo.getSoBienLai())
                    .ngayTao(LocalDateTime.now())
                    .build());
        }

        // 8. Persist batch (500/chunk)
        for (int i = 0; i < snapshots.size(); i += 500) {
            doiSoatHoSoRepository.saveAll(snapshots.subList(i, Math.min(i + 500, snapshots.size())));
        }

        log.info("[{}/TAO_DOI_SOAT] SUCCESS id={}, hoSoCount={}", user.getId(), doiSoat.getId(), snapshots.size());

        // 9. Return detail stub (full implementation in Task 02)
        return DoiSoatThuLaoDetailResponse.builder()
                .id(doiSoat.getId())
                .tenDoiSoat(doiSoat.getTenDoiSoat())
                .tuNgay(doiSoat.getTuNgay())
                .denNgay(doiSoat.getDenNgay())
                .trangThai(doiSoat.getTrangThai().name())
                .tenTrangThai(doiSoat.getTrangThai().getTen())
                .tongSoTienDong(snapshots.stream().map(NvDoiSoatThuLaoHoSo::getSoTienDong)
                        .reduce(BigDecimal.ZERO, BigDecimal::add))
                .tongThuLao(snapshots.stream().map(NvDoiSoatThuLaoHoSo::getThuLaoDuocHuong)
                        .reduce(BigDecimal.ZERO, BigDecimal::add))
                .build();
    }

    // ─── GET Đợt kê khai hợp lệ ────────────────────────────────────────────────
    public List<DotKeKhaiHopLeResponse> getDotKeKhaiHopLe(
            Integer daiLyThuId, LocalDate tuNgay, LocalDate denNgay) {

        AuthenticationService.getCurrentUserRequired();

        List<vn.vivas.bh.repository.projection.DotKeKhaiHopLeProjection> projections =
                doiSoatRepository.findDotKeKhaiHopLe(
                        daiLyThuId, tuNgay.toString(), denNgay.toString());

        return projections.stream()
                .map(p -> DotKeKhaiHopLeResponse.builder()
                        .dotKeKhaiId(p.getDotKeKhaiId())
                        .tenDotKeKhai(p.getTenDotKeKhai())
                        .thuTuc(p.getThuTuc())
                        .soLuongHoSoChuaDoiSoat(p.getSoLuongHoSoChuaDoiSoat())
                        .build())
                .toList();
    }
}
```

---

## 5. Controller

**Class**: `DoiSoatThuLaoController` | **Package**: `vn.vivas.bh.controller`

```java
package vn.vivas.bh.controller;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;
import vn.vivas.bh.constant.MessageResponseDict;
import vn.vivas.bh.request.doisoat.TaoDoiSoatThuLaoRequest;
import vn.vivas.bh.response.doisoat.*;
import vn.vivas.bh.service.DoiSoatThuLaoService;
import vn.vivas.bh.utils.ResponseUtils;

import java.time.LocalDate;
import java.util.List;

@Tag(name = "Đối Soát Thù Lao", description = "Quản lý kỳ đối soát thù lao thu hộ")
@RestController
@RequestMapping("/api/portal/v1/doi-soat-thu-lao")
@RequiredArgsConstructor
public class DoiSoatThuLaoController {

    private final DoiSoatThuLaoService service;

    @Operation(summary = "Danh sách kỳ đối soát")
    @GetMapping
    @PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.DOI_SOAT_THU_LAO)")
    public ResponseEntity<?> getDanhSach(
            @RequestParam(required = false) Integer daiLyThuId,
            @RequestParam(required = false) String trangThai,
            @RequestParam(required = false) String tenSearch,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate tuNgay,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate denNgay,
            @PageableDefault(size = 20) Pageable pageable) {

        Page<DoiSoatThuLaoListItemResponse> result =
                service.getDoiSoatList(daiLyThuId, trangThai, tenSearch, tuNgay, denNgay, pageable);
        return ResponseUtils.ok(MessageResponseDict.SUCCESS, result);
    }

    @Operation(summary = "Tạo kỳ đối soát mới")
    @PostMapping
    @PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.DOI_SOAT_THU_LAO)")
    public ResponseEntity<?> taoDoiSoat(@Valid @RequestBody TaoDoiSoatThuLaoRequest request) {
        DoiSoatThuLaoDetailResponse result = service.taoDoiSoatThuLao(request);
        return ResponseUtils.ok(MessageResponseDict.ADD_SUCCESS, result);
    }

    @Operation(summary = "Đợt kê khai hợp lệ để tạo kỳ đối soát")
    @GetMapping("/dot-ke-khai-hop-le")
    @PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.DOI_SOAT_THU_LAO)")
    public ResponseEntity<?> getDotKeKhaiHopLe(
            @RequestParam Integer daiLyThuId,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate tuNgay,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate denNgay) {

        List<DotKeKhaiHopLeResponse> result = service.getDotKeKhaiHopLe(daiLyThuId, tuNgay, denNgay);
        return ResponseUtils.ok(MessageResponseDict.SUCCESS, result);
    }
}
```

---

## 6. HoSoRepository — method cần thêm

Thêm vào `HoSoRepository` (existing class):

```java
// Package: vn.vivas.bh.repository.nghiepvu
// Thêm projection interface:

public interface HoSoForDoiSoatProjection {
    Integer getId();
    String getMaHoSo();
    Integer getDotKeKhaiId();
    String getLoaiNghiepVu();
    String getLoaiSanPham();
    java.math.BigDecimal getSoTienPhaiDong();
    java.math.BigDecimal getTiLeThuLao();        // NULL nếu chưa tính
    String getPhuongAnDong();
    String getPhuongThucDong();
    String getSoBienLai();
    String getTenNguoiThamGia();
    String getVung();                            // I|II|III từ dm_donvi_hanhchinh
}

// Thêm method:
@Query(value = """
    SELECT hs.id,
           hs.ma_ho_so           AS maHoSo,
           hs.dot_ke_khai_id     AS dotKeKhaiId,
           hs.loai_nghiep_vu     AS loaiNghiepVu,
           hs.loai_san_pham      AS loaiSanPham,
           hs.so_tien_phai_dong  AS soTienPhaiDong,
           hs.ti_le_thu_lao      AS tiLeThuLao,
           hs.phuong_an_dong     AS phuongAnDong,
           hs.phuong_thuc_dong   AS phuongThucDong,
           hs.so_bien_lai        AS soBienLai,
           ntt.ho_ten            AS tenNguoiThamGia,
           dvhc.ma_vung          AS vung
    FROM nv_ho_so_dang_ky hs
    LEFT JOIN nv_nguoi_tham_gia ntt ON ntt.ho_so_id = hs.id
    LEFT JOIN dm_donvi_hanhchinh dvhc
           ON dvhc.ma = hs.tinh_tp_cu_tru AND dvhc.cap = 'tinh'
    WHERE hs.dot_ke_khai_id IN :dotKeKhaiIds
      AND hs.trang_thai IN ('THANH_CONG', 'CQ_BHXH_DANG_XU_LY')
    """, nativeQuery = true)
List<HoSoForDoiSoatProjection> findAllByDotKeKhaiIdIn(@Param("dotKeKhaiIds") List<Integer> dotKeKhaiIds);
```

---

## 7. Unit Tests

**Class**: `DoiSoatThuLaoServiceCreateTest` | **Package**: `vn.vivas.bh.service`

```java
package vn.vivas.bh.service;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import vn.vivas.bh.auth.AccountInfo;
import vn.vivas.bh.auth.AuthenticationService;
import vn.vivas.bh.constant.MessageResponseDict;
import vn.vivas.bh.exception.BusinessException;
import vn.vivas.bh.repository.nghiepvu.DoiSoatThuLaoHoSoRepository;
import vn.vivas.bh.repository.nghiepvu.DoiSoatThuLaoRepository;
import vn.vivas.bh.repository.nghiepvu.HoSoRepository;
import vn.vivas.bh.request.doisoat.TaoDoiSoatThuLaoRequest;

import java.time.LocalDate;
import java.util.Arrays;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class DoiSoatThuLaoServiceCreateTest {

    @Mock private DoiSoatThuLaoRepository doiSoatRepository;
    @Mock private DoiSoatThuLaoHoSoRepository doiSoatHoSoRepository;
    @Mock private HoSoRepository hoSoRepository;
    @InjectMocks private DoiSoatThuLaoService service;

    @BeforeEach
    void setUp() {
        AccountInfo user = AccountInfo.builder()
                .id(1).role(vn.vivas.bh.enums.Role.ADMIN)
                .permissions(Arrays.asList("ALL_PERMISSIONS")).daiLyId(null).build();
        when(AuthenticationService.getCurrentUserRequired()).thenReturn(user);
    }

    @Test
    void shouldThrowKhiTenDoiSoatTrungLap() {
        TaoDoiSoatThuLaoRequest rq = new TaoDoiSoatThuLaoRequest();
        rq.setTenDoiSoat("Kỳ A"); rq.setTuNgay(LocalDate.now().minusMonths(1));
        rq.setDenNgay(LocalDate.now()); rq.setDotKeKhaiIds(List.of(1));

        when(doiSoatRepository.existsByTenDoiSoat("Kỳ A")).thenReturn(true);

        assertThatThrownBy(() -> service.taoDoiSoatThuLao(rq))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining(MessageResponseDict.DOI_SOAT_TEN_DUPLICATE.getMessage());
    }

    @Test
    void shouldThrowKhiDenNgayTuongLai() {
        TaoDoiSoatThuLaoRequest rq = new TaoDoiSoatThuLaoRequest();
        rq.setTenDoiSoat("Kỳ B"); rq.setTuNgay(LocalDate.now());
        rq.setDenNgay(LocalDate.now().plusDays(1)); rq.setDotKeKhaiIds(List.of(1));

        when(doiSoatRepository.existsByTenDoiSoat(anyString())).thenReturn(false);

        assertThatThrownBy(() -> service.taoDoiSoatThuLao(rq))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining(MessageResponseDict.INVALID_DATE_RANGE.getMessage());
    }

    @Test
    void shouldThrowKhiKhongChonDotKeKhai() {
        TaoDoiSoatThuLaoRequest rq = new TaoDoiSoatThuLaoRequest();
        rq.setTenDoiSoat("Kỳ C"); rq.setTuNgay(LocalDate.now().minusMonths(1));
        rq.setDenNgay(LocalDate.now()); rq.setDotKeKhaiIds(List.of());

        when(doiSoatRepository.existsByTenDoiSoat(anyString())).thenReturn(false);

        assertThatThrownBy(() -> service.taoDoiSoatThuLao(rq))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining(MessageResponseDict.DOI_SOAT_EMPTY_DOT.getMessage());
    }
}
```

---

## 8. Checklist

- [ ] `TaoDoiSoatThuLaoRequest` — @NotBlank, @NotNull, @Size(min=1)
- [ ] `DoiSoatThuLaoListItemResponse`, `DoiSoatThuLaoDetailResponse`, `DoiSoatHoSoDetailItem` — @Builder
- [ ] `DotKeKhaiHopLeResponse` — @Builder
- [ ] `DoiSoatThuLaoService.getDoiSoatList()` — RBAC scope, native query, page map
- [ ] `DoiSoatThuLaoService.taoDoiSoatThuLao()` — 9 steps, batch load, snapshot, saveAll batch 500
- [ ] `DoiSoatThuLaoService.getDotKeKhaiHopLe()` — query + map
- [ ] `HoSoRepository.findAllByDotKeKhaiIdIn()` — native query + `HoSoForDoiSoatProjection`
- [ ] `DoiSoatThuLaoController` — 3 endpoints, @PreAuthorize, @Valid
- [ ] Unit tests — tên duplicate, date future, dotKeKhai empty
- [ ] `./mvnw test` — pass

## 9. Sync Points

**Depends on**: Task 00 (entities, repos, enums, error codes, permission)  
**Provides to Task 02**: `DoiSoatThuLaoService` class (Task 02 thêm methods vào)  
**Provides to Task 02**: `DoiSoatHoSoDetailItem`, `DoiSoatThuLaoDetailResponse` responses
