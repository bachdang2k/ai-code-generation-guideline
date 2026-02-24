# Task 02: Detail + Chốt + Sửa + Xóa — Đối Soát Thù Lao

**PAKT**: §2, §4, §6 (chotDoiSoat, suaDoiSoat, xoaDoiSoat), §8 (Security)  
**Hours**: 10  
**Dependencies**: Task 00 ✅, Task 01 ✅ (DoiSoatThuLaoService class đã tạo)

---

## 1. API Contract

| Method | Path | Request | Response | Errors |
|--------|------|---------|----------|--------|
| GET | `/api/portal/v1/doi-soat-thu-lao/{id}` | PathVar id, Pageable | `DoiSoatThuLaoDetailResponse` | DOI_SOAT_NOT_FOUND, DOI_SOAT_FORBIDDEN |
| POST | `/api/portal/v1/doi-soat-thu-lao/{id}/chot` | PathVar id | `void` | DOI_SOAT_NOT_FOUND, DOI_SOAT_INVALID_STATE, DOI_SOAT_FORBIDDEN |
| PUT | `/api/portal/v1/doi-soat-thu-lao/{id}` | `SuaDoiSoatThuLaoRequest` | `DoiSoatThuLaoDetailResponse` | DOI_SOAT_NOT_FOUND, DOI_SOAT_INVALID_STATE, DOI_SOAT_FORBIDDEN |
| DELETE | `/api/portal/v1/doi-soat-thu-lao/{id}` | PathVar id | `void` | DOI_SOAT_NOT_FOUND, DOI_SOAT_INVALID_STATE, DOI_SOAT_FORBIDDEN |
| GET | `/api/portal/v1/doi-soat-thu-lao/{id}/ho-so-le` | PathVar id, keyword, Pageable | `Page<HoSoLeResponse>` | DOI_SOAT_NOT_FOUND |

---

## 2. Request DTO

**Package**: `vn.vivas.bh.request.doisoat`

```java
package vn.vivas.bh.request.doisoat;

import lombok.Data;
import java.util.List;

/** Sửa kỳ đối soát: thêm/xóa hồ sơ lẻ */
@Data
public class SuaDoiSoatThuLaoRequest {
    private List<Integer> themHoSoIds;  // nullable — hồ sơ lẻ thêm vào
    private List<Integer> xoaHoSoIds;  // nullable — hồ sơ cần bỏ khỏi kỳ
}
```

---

## 3. Response DTO — HoSoLeResponse

**Package**: `vn.vivas.bh.response.doisoat`

```java
package vn.vivas.bh.response.doisoat;

import lombok.Builder;
import lombok.Data;

@Data @Builder
public class HoSoLeResponse {
    private Integer hoSoId;
    private String maHoSo;
    private String tenDotKeKhai;
    private String tenNguoiThamGia;
    private String thuTuc;    // BHXH_TU_NGUYEN | BHYT_HO_GIA_DINH
    private String trangThai;
}
```

---

## 3.5. Projection + Repository Method — Aggregate Totals

**Package projection**: `vn.vivas.bh.repository.projection`

```java
package vn.vivas.bh.repository.projection;

import java.math.BigDecimal;

/**
 * Native SQL projection trả về tổng tiền đóng và tổng thù lao
 * cho 1 kỳ đối soát. Dùng thay cho load-all + Java stream reduce.
 */
public interface DoiSoatThuLaoTotalsProjection {
    BigDecimal getTongSoTienDong();
    BigDecimal getTongThuLao();
}
```

**Repository method** — thêm vào `DoiSoatThuLaoHoSoRepository`:

```java
/**
 * Tính tổng trực tiếp trên DB — O(1) network, tránh load N rows vào heap.
 * Trả về NULL nếu kỳ chưa có hồ sơ nào → service phải coalesce về ZERO.
 */
@Query(value = """
    SELECT
        SUM(so_tien_dong)       AS tongSoTienDong,
        SUM(thu_lao_duoc_huong) AS tongThuLao
    FROM nv_doi_soat_thu_lao_ho_so
    WHERE doi_soat_thu_lao_id = :doiSoatThuLaoId
      AND deleted_at IS NULL
    """, nativeQuery = true)
DoiSoatThuLaoTotalsProjection sumTotals(@Param("doiSoatThuLaoId") Integer doiSoatThuLaoId);
```

> **Lý do native query**: `SUM` trả về `BigDecimal` qua JDBC projection — JPA nativeQuery
> mapping hoạt động chính xác hơn JPQL khi cần alias khớp tên getter projection.

---

## 4. Service Methods — thêm vào `DoiSoatThuLaoService`

```java
// Thêm vào class DoiSoatThuLaoService (đã tạo ở Task 01)

// ─── GET Chi tiết kỳ đối soát ───────────────────────────────────────────────
public DoiSoatThuLaoDetailResponse getDoiSoatDetail(Integer id, Pageable pageable) {
    // 1. Auth + load
    AccountInfo user = AuthenticationService.getCurrentUserRequired();
    NvDoiSoatThuLao doiSoat = doiSoatRepository.findById(id)
            .orElseThrow(() -> new BusinessException(MessageResponseDict.DOI_SOAT_NOT_FOUND));

    // 2. RBAC ownership check
    kiemTraQuyenSoHuu(doiSoat, user);

    // 3. Load hồ sơ con — phân trang
    Page<NvDoiSoatThuLaoHoSo> hoSoPage =
            doiSoatHoSoRepository.findAllByDoiSoatThuLaoId(id, pageable);

    // 4. Aggregate totals — native SQL SUM (tránh load toàn bộ rows vào memory)
    // Repository method: DoiSoatThuLaoHoSoRepository.sumTotals(id) → DoiSoatThuLaoTotalsProjection
    DoiSoatThuLaoTotalsProjection totals =
            doiSoatHoSoRepository.sumTotals(id);
    BigDecimal tongSoTienDong = totals.getTongSoTienDong() != null
            ? totals.getTongSoTienDong() : BigDecimal.ZERO;
    BigDecimal tongThuLao = totals.getTongThuLao() != null
            ? totals.getTongThuLao() : BigDecimal.ZERO;

    // 5. Map hồ sơ — resolve phuongAnDong/phuongThucDong bằng Java map (load-once)
    // Note: Java key-value map implementation — load từ dm tables, cache per request scope
    Page<DoiSoatHoSoDetailItem> hoSoItems = hoSoPage.map(h -> DoiSoatHoSoDetailItem.builder()
            .id(h.getId())
            .maHoSo(h.getMaHoSo())
            .tenNguoiThamGia(h.getTenNguoiThamGia())
            .thuTuc(resolveThuTuc(h.getLoaiNghiepVu()))
            .vung(h.getVung())
            .phuongAnDong(h.getPhuongAnDong())   // mã — FE tự display hoặc service resolve
            .phuongThucDong(h.getPhuongThucDong())
            .soBienLai(h.getSoBienLai())
            .soTienDong(h.getSoTienDong())
            .tiLeThuLao(h.getTiLeThuLao())
            .thuLaoDuocHuong(h.getThuLaoDuocHuong())
            .build());

    // 6. Build agency/cq names (JOIN already done in load nếu cần — hoặc load riêng)
    return DoiSoatThuLaoDetailResponse.builder()
            .id(doiSoat.getId())
            .tenDoiSoat(doiSoat.getTenDoiSoat())
            .tuNgay(doiSoat.getTuNgay())
            .denNgay(doiSoat.getDenNgay())
            .trangThai(doiSoat.getTrangThai().name())
            .tenTrangThai(doiSoat.getTrangThai().getTen())
            .hoSoList(hoSoItems)
            .tongSoTienDong(tongSoTienDong)
            .tongThuLao(tongThuLao)
            .build();
}

// ─── POST Chốt kỳ đối soát ─────────────────────────────────────────────────
@Transactional
public void chotDoiSoatThuLao(Integer id) {
    // 1. Auth
    AccountInfo user = AuthenticationService.getCurrentUserRequired();
    log.info("[{}/CHOT_DOI_SOAT] id={}", user.getId(), id);

    // 2. Load
    NvDoiSoatThuLao doiSoat = doiSoatRepository.findById(id)
            .orElseThrow(() -> new BusinessException(MessageResponseDict.DOI_SOAT_NOT_FOUND));

    // 3. RBAC
    kiemTraQuyenSoHuu(doiSoat, user);

    // 4. State validate
    DoiSoatThuLaoStateMachine.validateCoTheThaoTac(doiSoat.getTrangThai());

    // 5. Transition
    doiSoat.setTrangThai(DoiSoatThuLaoStateMachine.chotDoiSoat(doiSoat.getTrangThai()));
    doiSoat.setNguoiChotDoiSoat(user.getId());
    doiSoat.setNgayChotDoiSoat(LocalDateTime.now());
    doiSoatRepository.save(doiSoat);

    log.info("[{}/CHOT_DOI_SOAT] SUCCESS id={}", user.getId(), id);
}

// ─── PUT Sửa kỳ đối soát ──────────────────────────────────────────────────
@Transactional
public DoiSoatThuLaoDetailResponse suaDoiSoatThuLao(Integer id, SuaDoiSoatThuLaoRequest request) {
    // 1. Auth
    AccountInfo user = AuthenticationService.getCurrentUserRequired();
    log.info("[{}/SUA_DOI_SOAT] id={}", user.getId(), id);

    // 2. Load + RBAC + state
    NvDoiSoatThuLao doiSoat = doiSoatRepository.findById(id)
            .orElseThrow(() -> new BusinessException(MessageResponseDict.DOI_SOAT_NOT_FOUND));
    kiemTraQuyenSoHuu(doiSoat, user);
    DoiSoatThuLaoStateMachine.validateCoTheThaoTac(doiSoat.getTrangThai());

    // 3. Xóa hồ sơ lẻ (nếu có)
    if (request.getXoaHoSoIds() != null && !request.getXoaHoSoIds().isEmpty()) {
        int deleted = doiSoatHoSoRepository
                .deleteByDoiSoatThuLaoIdAndHoSoIdIn(id, request.getXoaHoSoIds());
        log.info("[{}/SUA_DOI_SOAT] xoa hoSo count={}", user.getId(), deleted);
    }

    // 4. Thêm hồ sơ lẻ (nếu có)
    if (request.getThemHoSoIds() != null && !request.getThemHoSoIds().isEmpty()) {
        // Validate chưa thuộc kỳ nào — batch check
        List<Integer> daDuocDoiSoat = doiSoatHoSoRepository
                .findHoSoIdsDaDoiSoat(request.getThemHoSoIds());
        if (!daDuocDoiSoat.isEmpty()) {
            log.warn("[{}/SUA_DOI_SOAT] hoSo ids already in reconciliation: {}", user.getId(), daDuocDoiSoat);
            // Filter out — chỉ thêm những hồ sơ hợp lệ
        }

        List<Integer> hopLeIds = request.getThemHoSoIds().stream()
                .filter(hoSoId -> !daDuocDoiSoat.contains(hoSoId))
                .toList();

        if (!hopLeIds.isEmpty()) {
            List<HoSoForDoiSoatProjection> newHoSoList =
                    hoSoRepository.findAllByDotKeKhaiIdIn(hopLeIds); // reuse projection
            // Build snapshots
            List<NvDoiSoatThuLaoHoSo> newSnapshots = new ArrayList<>();
            for (HoSoForDoiSoatProjection hoSo : newHoSoList) {
                BigDecimal tiLe = hoSo.getTiLeThuLao() != null ? hoSo.getTiLeThuLao() : BigDecimal.ZERO;
                BigDecimal soTien = hoSo.getSoTienPhaiDong() != null ? hoSo.getSoTienPhaiDong() : BigDecimal.ZERO;
                BigDecimal thuLao = soTien.multiply(tiLe).divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_UP);

                newSnapshots.add(NvDoiSoatThuLaoHoSo.builder()
                        .doiSoatThuLaoId(id)
                        .hoSoId(hoSo.getId())
                        .dotKeKhaiId(hoSo.getDotKeKhaiId())
                        .soTienDong(soTien).tiLeThuLao(tiLe).thuLaoDuocHuong(thuLao)
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
            doiSoatHoSoRepository.saveAll(newSnapshots);
            log.info("[{}/SUA_DOI_SOAT] them hoSo count={}", user.getId(), newSnapshots.size());
        }
    }

    log.info("[{}/SUA_DOI_SOAT] SUCCESS id={}", user.getId(), id);
    return getDoiSoatDetail(id, Pageable.ofSize(20));
}

// ─── DELETE Xóa kỳ đối soát ────────────────────────────────────────────────
@Transactional
public void xoaDoiSoatThuLao(Integer id) {
    // 1. Auth
    AccountInfo user = AuthenticationService.getCurrentUserRequired();
    log.info("[{}/XOA_DOI_SOAT] id={}", user.getId(), id);

    // 2. Load + RBAC + state
    NvDoiSoatThuLao doiSoat = doiSoatRepository.findById(id)
            .orElseThrow(() -> new BusinessException(MessageResponseDict.DOI_SOAT_NOT_FOUND));
    kiemTraQuyenSoHuu(doiSoat, user);
    DoiSoatThuLaoStateMachine.validateCoTheThaoTac(doiSoat.getTrangThai());

    // 3. Clear hồ sơ con (revert hồ sơ về trạng thái available)
    int reverted = doiSoatHoSoRepository.deleteAllByDoiSoatThuLaoId(id);

    // 4. Soft delete kỳ
    doiSoatRepository.delete(doiSoat);  // @SQLDelete → sets deleted_at = NOW()

    log.info("[{}/XOA_DOI_SOAT] SUCCESS id={}, hoSoReverted={}", user.getId(), id, reverted);
}

// ─── GET Hồ sơ lẻ có thể thêm ──────────────────────────────────────────────
public Page<HoSoLeResponse> getHoSoLe(Integer doiSoatId, String keyword, Pageable pageable) {
    // 1. Auth
    AccountInfo user = AuthenticationService.getCurrentUserRequired();

    // 2. Load kỳ đối soát để lấy daiLyThuId
    NvDoiSoatThuLao doiSoat = doiSoatRepository.findById(doiSoatId)
            .orElseThrow(() -> new BusinessException(MessageResponseDict.DOI_SOAT_NOT_FOUND));

    // 3. Query hồ sơ lẻ (chưa thuộc kỳ nào, của đại lý này)
    Page<vn.vivas.bh.repository.projection.HoSoLeProjection> projections =
            doiSoatHoSoRepository.findHoSoLe(doiSoat.getDaiLyThuId(), keyword, pageable);

    // 4. Map
    return projections.map(p -> HoSoLeResponse.builder()
            .hoSoId(p.getHoSoId())
            .maHoSo(p.getMaHoSo())
            .tenDotKeKhai(p.getTenDotKeKhai())
            .tenNguoiThamGia(p.getTenNguoiThamGia())
            .thuTuc(p.getThuTuc())
            .trangThai(p.getTrangThai())
            .build());
}

// ─── Private Helpers ────────────────────────────────────────────────────────

/** RBAC: Admin bypass, Đại lý thu chỉ thao tác own kỳ */
private void kiemTraQuyenSoHuu(NvDoiSoatThuLao doiSoat, AccountInfo user) {
    if (user.isAdmin()) return;
    if (!doiSoat.getCreatedByUserId().equals(user.getId())) {
        throw new BusinessException(MessageResponseDict.DOI_SOAT_FORBIDDEN);
    }
}

/** Resolve loaiNghiepVu → display label */
private String resolveThuTuc(String loaiNghiepVu) {
    if ("BHXH_TU_NGUYEN".equals(loaiNghiepVu)) return "BHXH Tự nguyện";
    if ("BHYT_HO_GIA_DINH".equals(loaiNghiepVu)) return "BHYT Hộ gia đình";
    return loaiNghiepVu;
}
```

---

## 5. Controller — thêm endpoints vào `DoiSoatThuLaoController`

```java
// Thêm vào DoiSoatThuLaoController (đã có ở Task 01)

@Operation(summary = "Xem chi tiết kỳ đối soát")
@GetMapping("/{id}")
@PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.DOI_SOAT_THU_LAO)")
public ResponseEntity<?> getChiTiet(
        @PathVariable Integer id,
        @PageableDefault(size = 20) Pageable pageable) {
    DoiSoatThuLaoDetailResponse result = service.getDoiSoatDetail(id, pageable);
    return ResponseUtils.ok(MessageResponseDict.SUCCESS, result);
}

@Operation(summary = "Chốt kỳ đối soát")
@PostMapping("/{id}/chot")
@PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.DOI_SOAT_THU_LAO)")
public ResponseEntity<?> chotDoiSoat(@PathVariable Integer id) {
    service.chotDoiSoatThuLao(id);
    return ResponseUtils.ok(MessageResponseDict.SUCCESS, null);
}

@Operation(summary = "Sửa kỳ đối soát (thêm/xóa hồ sơ lẻ)")
@PutMapping("/{id}")
@PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.DOI_SOAT_THU_LAO)")
public ResponseEntity<?> suaDoiSoat(
        @PathVariable Integer id,
        @RequestBody SuaDoiSoatThuLaoRequest request) {
    DoiSoatThuLaoDetailResponse result = service.suaDoiSoatThuLao(id, request);
    return ResponseUtils.ok(MessageResponseDict.UPDATE_SUCCESS, result);
}

@Operation(summary = "Xóa kỳ đối soát")
@DeleteMapping("/{id}")
@PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.DOI_SOAT_THU_LAO)")
public ResponseEntity<?> xoaDoiSoat(@PathVariable Integer id) {
    service.xoaDoiSoatThuLao(id);
    return ResponseUtils.ok(MessageResponseDict.DELETE_SUCCESS, null);
}

@Operation(summary = "Hồ sơ lẻ có thể thêm vào kỳ")
@GetMapping("/{id}/ho-so-le")
@PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.DOI_SOAT_THU_LAO)")
public ResponseEntity<?> getHoSoLe(
        @PathVariable Integer id,
        @RequestParam(required = false) String keyword,
        @PageableDefault(size = 20) Pageable pageable) {
    Page<HoSoLeResponse> result = service.getHoSoLe(id, keyword, pageable);
    return ResponseUtils.ok(MessageResponseDict.SUCCESS, result);
}
```

---

## 6. Unit Tests

**Class**: `DoiSoatThuLaoServiceDetailTest` | **Package**: `vn.vivas.bh.service`

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
import vn.vivas.bh.entity.nghiepvu.NvDoiSoatThuLao;
import vn.vivas.bh.enums.Role;
import vn.vivas.bh.enums.TrangThaiDoiSoat;
import vn.vivas.bh.exception.BusinessException;
import vn.vivas.bh.repository.nghiepvu.DoiSoatThuLaoHoSoRepository;
import vn.vivas.bh.repository.nghiepvu.DoiSoatThuLaoRepository;
import vn.vivas.bh.repository.nghiepvu.HoSoRepository;

import java.time.LocalDate;
import java.util.Arrays;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class DoiSoatThuLaoServiceDetailTest {

    @Mock private DoiSoatThuLaoRepository doiSoatRepository;
    @Mock private DoiSoatThuLaoHoSoRepository doiSoatHoSoRepository;
    @Mock private HoSoRepository hoSoRepository;
    @InjectMocks private DoiSoatThuLaoService service;

    private NvDoiSoatThuLao doiSoatDaDuyet;
    private NvDoiSoatThuLao doiSoatDangXuLy;

    @BeforeEach
    void setUp() {
        AccountInfo admin = AccountInfo.builder()
                .id(1).role(Role.ADMIN)
                .permissions(Arrays.asList("ALL_PERMISSIONS")).daiLyId(null).build();
        when(AuthenticationService.getCurrentUserRequired()).thenReturn(admin);

        doiSoatDaDuyet = NvDoiSoatThuLao.builder()
                .id(1).tenDoiSoat("Kỳ đã chốt")
                .tuNgay(LocalDate.now().minusMonths(1)).denNgay(LocalDate.now())
                .trangThai(TrangThaiDoiSoat.DA_DOI_SOAT).createdByUserId(1).build();

        doiSoatDangXuLy = NvDoiSoatThuLao.builder()
                .id(2).tenDoiSoat("Kỳ đang xử lý")
                .tuNgay(LocalDate.now().minusMonths(1)).denNgay(LocalDate.now())
                .trangThai(TrangThaiDoiSoat.DANG_DOI_SOAT).createdByUserId(1).build();
    }

    @Test
    void shouldThrowKhiChotKyDaChot() {
        when(doiSoatRepository.findById(1)).thenReturn(Optional.of(doiSoatDaDuyet));

        assertThatThrownBy(() -> service.chotDoiSoatThuLao(1))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining(MessageResponseDict.DOI_SOAT_INVALID_STATE.getMessage());
    }

    @Test
    void shouldThrowKhiXoaKyDaChot() {
        when(doiSoatRepository.findById(1)).thenReturn(Optional.of(doiSoatDaDuyet));

        assertThatThrownBy(() -> service.xoaDoiSoatThuLao(1))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining(MessageResponseDict.DOI_SOAT_INVALID_STATE.getMessage());
    }

    @Test
    void shouldThrowWhenDoiSoatNotFound() {
        when(doiSoatRepository.findById(999)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> service.chotDoiSoatThuLao(999))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining(MessageResponseDict.DOI_SOAT_NOT_FOUND.getMessage());
    }

    @Test
    void shouldThrowKhiDaiLyThuChotKyCuaNguoiKhac() {
        // Setup: user là Đại lý thu (không phải admin), kỳ được tạo bởi user khác
        AccountInfo daiLyThu = AccountInfo.builder()
                .id(99).role(Role.DAI_LY_THU)  // không phải admin
                .permissions(Arrays.asList("DOI_SOAT_THU_LAO")).daiLyId(5).build();
        when(AuthenticationService.getCurrentUserRequired()).thenReturn(daiLyThu);

        NvDoiSoatThuLao kyNguoiKhac = NvDoiSoatThuLao.builder()
                .id(5).trangThai(TrangThaiDoiSoat.DANG_DOI_SOAT).createdByUserId(1).build();
        when(doiSoatRepository.findById(5)).thenReturn(Optional.of(kyNguoiKhac));

        assertThatThrownBy(() -> service.chotDoiSoatThuLao(5))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining(MessageResponseDict.DOI_SOAT_FORBIDDEN.getMessage());
    }
}
```

---

## 7. Checklist

- [ ] `SuaDoiSoatThuLaoRequest` — themHoSoIds, xoaHoSoIds (nullable)
- [ ] `HoSoLeResponse` — @Builder
- [ ] `DoiSoatThuLaoService.getDoiSoatDetail()` — load + RBAC + page map + totals
- [ ] `DoiSoatThuLaoService.chotDoiSoatThuLao()` — load + RBAC + state machine + save
- [ ] `DoiSoatThuLaoService.suaDoiSoatThuLao()` — batch delete + batch add snapshots
- [ ] `DoiSoatThuLaoService.xoaDoiSoatThuLao()` — deleteAll + soft delete
- [ ] `DoiSoatThuLaoService.getHoSoLe()` — load kỳ + query repo + map
- [ ] `DoiSoatThuLaoService.kiemTraQuyenSoHuu()` — Admin bypass, Đại lý thu ownership check
- [ ] Controller: 5 endpoints mới với @PreAuthorize
- [ ] Unit tests — chốt kỳ đã chốt, xóa kỳ đã chốt, not found, RBAC forbidden
- [ ] `./mvnw test` — pass

## 8. Sync Points

**Depends on**:
- Task 00: `NvDoiSoatThuLao`, `NvDoiSoatThuLaoHoSo`, `DoiSoatThuLaoStateMachine`, error codes
- Task 01: `DoiSoatThuLaoService` (phải thêm methods vào class này), `DoiSoatHoSoDetailItem`, `DoiSoatThuLaoDetailResponse`

**Provides to Task 03**:
- `DoiSoatThuLaoService` class (Task 03 thêm export methods vào đây, hoặc inject `DoiSoatExportService`)
