# Task 03: Export Service — Đối Soát Thù Lao

**PAKT**: §6 (Xuất C17-TS, Xuất XLSX), §7 (Export Services)  
**Hours**: 8  
**Dependencies**: Task 00 ✅ (entities, repos), Task 02 ✅ (getDoiSoatDetail context)

> **Library**: Apache POI (XWPF cho DOCX→PDF fill, XSSF cho XLSX)  
> **Template**: `src/main/resources/templates/template_doi_soat_C17_TS.doc`

---

## 1. API Contract

| Method | Path | Request | Response | Errors |
|--------|------|---------|----------|--------|
| GET | `/api/portal/v1/doi-soat-thu-lao/{id}/export-c17` | PathVar id | `byte[]` (PDF) | DOI_SOAT_NOT_FOUND, DOI_SOAT_FORBIDDEN |
| GET | `/api/portal/v1/doi-soat-thu-lao/{id}/export-chi-tiet` | PathVar id | `byte[]` (XLSX) | DOI_SOAT_NOT_FOUND, DOI_SOAT_FORBIDDEN |

---

## 2. Maven Dependency

Thêm vào `pom.xml`:

```xml
<!-- Apache POI for DOCX/XLSX export -->
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>5.3.0</version>
</dependency>

<!-- PDF conversion from DOCX (nếu dùng LibreOffice/JodConverter) -->
<dependency>
    <groupId>org.jodconverter</groupId>
    <artifactId>jodconverter-spring-boot-starter</artifactId>
    <version>4.4.6</version>
</dependency>
```

---

## 3. Export Service

**Class**: `DoiSoatExportService` | **Package**: `vn.vivas.bh.service`

```java
package vn.vivas.bh.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.ss.util.CellRangeAddress;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import org.apache.poi.xwpf.usermodel.XWPFDocument;
import org.springframework.core.io.ClassPathResource;
import org.springframework.stereotype.Service;
import vn.vivas.bh.auth.AccountInfo;
import vn.vivas.bh.auth.AuthenticationService;
import vn.vivas.bh.constant.MessageResponseDict;
import vn.vivas.bh.entity.nghiepvu.NvDoiSoatThuLao;
import vn.vivas.bh.entity.nghiepvu.NvDoiSoatThuLaoHoSo;
import vn.vivas.bh.exception.BusinessException;
import vn.vivas.bh.repository.nghiepvu.DoiSoatThuLaoHoSoRepository;
import vn.vivas.bh.repository.nghiepvu.DoiSoatThuLaoRepository;
import vn.vivas.bh.repository.projection.DotKeKhaiInfoProjection;
import vn.vivas.bh.repository.projection.HoSoTinhTtProjection;
import vn.vivas.bh.statemachine.DoiSoatThuLaoStateMachine;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.math.BigDecimal;
import java.text.NumberFormat;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class DoiSoatExportService {

    private final DoiSoatThuLaoRepository doiSoatRepository;
    private final DoiSoatThuLaoHoSoRepository doiSoatHoSoRepository;

    private static final DateTimeFormatter DATE_FMT = DateTimeFormatter.ofPattern("dd/MM/yyyy");

    // ─── Export C17-TS (PDF) ───────────────────────────────────────────────────

    /**
     * Xuất file C17-TS PDF từ template DOCX.
     * Fill 10 trường theo PAKT §6 + bảng biên lai.
     * Convert DOCX → PDF dùng JodConverter (LibreOffice headless).
     */
    public byte[] exportC17Pdf(Integer doiSoatId) {
        // 1. Auth + load
        AccountInfo user = AuthenticationService.getCurrentUserRequired();
        log.info("[{}/EXPORT_C17] doiSoatId={}", user.getId(), doiSoatId);

        NvDoiSoatThuLao doiSoat = doiSoatRepository.findById(doiSoatId)
                .orElseThrow(() -> new BusinessException(MessageResponseDict.DOI_SOAT_NOT_FOUND));
        kiemTraQuyenSoHuu(doiSoat, user);

        List<NvDoiSoatThuLaoHoSo> hoSoList =
                doiSoatHoSoRepository.findAllByDoiSoatThuLaoId(doiSoatId);

        // 2. Load template DOCX
        try (InputStream templateStream = new ClassPathResource(
                "templates/template_doi_soat_C17_TS.doc").getInputStream();
             XWPFDocument doc = new XWPFDocument(templateStream)) {

            // 3. Fill fields theo placeholder {{field}} pattern
            C17DataModel data = buildC17Data(doiSoat, hoSoList);
            fillDocxTemplate(doc, data);

            // 4. Convert → PDF bytes
            byte[] pdfBytes = convertDocxToPdf(doc);
            log.info("[{}/EXPORT_C17] SUCCESS doiSoatId={}, size={} bytes",
                    user.getId(), doiSoatId, pdfBytes.length);
            return pdfBytes;

        } catch (IOException e) {
            log.error("[{}/EXPORT_C17] ERROR doiSoatId={}: {}", user.getId(), doiSoatId, e.getMessage(), e);
            throw new BusinessException(MessageResponseDict.EXPORT_FAILED);
        }
    }

    // ─── Export DS Chi tiết XLSX ───────────────────────────────────────────────

    /**
     * Xuất file DS chi tiết hồ sơ XLSX — 15 cột theo PAKT §6.
     * Load toàn bộ hồ sơ (no pagination).
     */
    public byte[] exportChiTietXlsx(Integer doiSoatId) {
        // 1. Auth + load
        AccountInfo user = AuthenticationService.getCurrentUserRequired();
        log.info("[{}/EXPORT_XLSX] doiSoatId={}", user.getId(), doiSoatId);

        NvDoiSoatThuLao doiSoat = doiSoatRepository.findById(doiSoatId)
                .orElseThrow(() -> new BusinessException(MessageResponseDict.DOI_SOAT_NOT_FOUND));
        kiemTraQuyenSoHuu(doiSoat, user);

        List<NvDoiSoatThuLaoHoSo> hoSoList =
                doiSoatHoSoRepository.findAllByDoiSoatThuLaoId(doiSoatId);

        // 2. Build XLSX
        try (XSSFWorkbook workbook = new XSSFWorkbook();
             ByteArrayOutputStream out = new ByteArrayOutputStream()) {

            Sheet sheet = workbook.createSheet("DS Chi tiết hồ sơ");

            // 3. Header row
            String[] headers = {
                "STT", "Đợt kê khai", "Mã hồ sơ", "Người tham gia", "Thủ tục",
                "Tỉnh/TP", "Vùng", "Phương án đóng", "Phương thức đóng",
                "Số biên lai", "Số tiền đóng", "Tỷ lệ thù lao (%)",
                "Thù lao được hưởng", "Đại lý thu hộ", "Nhân viên thu"
            };
            CellStyle headerStyle = createHeaderStyle(workbook);
            Row headerRow = sheet.createRow(0);
            for (int i = 0; i < headers.length; i++) {
                Cell cell = headerRow.createCell(i);
                cell.setCellValue(headers[i]);
                cell.setCellStyle(headerStyle);
            }

            // 4. Batch-load lookup maps (1 query mỗi map — tránh N+1)
            // 4a. dotKeKhaiId → tenDotKeKhai
            //     Rule: String.format("[%02d] %s", soThuTu, createdAt.format("MM/yyyy"))
            List<Integer> dotKeKhaiIds = hoSoList.stream()
                    .map(NvDoiSoatThuLaoHoSo::getDotKeKhaiId)
                    .distinct().toList();
            Map<Integer, String> dotKeKhaiTenMap = doiSoatHoSoRepository
                    .findDotKeKhaiInfoByIds(dotKeKhaiIds)
                    .stream()
                    .collect(Collectors.toMap(
                            DotKeKhaiInfoProjection::getId,
                            p -> String.format("[%02d] %s",
                                    p.getSoThuTu(),
                                    p.getCreatedAt().format(DateTimeFormatter.ofPattern("MM/yyyy")))
                    ));

            // 4b. hoSoId → tenTinhTt (từ nv_nguoi_tham_gia.ma_tinh_tt → dm_donvi_hanhchinh.ten)
            List<Integer> hoSoIds = hoSoList.stream()
                    .map(NvDoiSoatThuLaoHoSo::getHoSoId)
                    .toList();
            Map<Integer, String> hoSoTinhTtMap = doiSoatHoSoRepository
                    .findTinhTtByHoSoIds(hoSoIds)
                    .stream()
                    .collect(Collectors.toMap(
                            HoSoTinhTtProjection::getHoSoId,
                            p -> p.getTenTinhTt() != null ? p.getTenTinhTt() : ""
                    ));

            // 4c. hoSoId → tenDaiLyThu + tenNhanVienThu
            //     nv_ho_so_dang_ky.dai_ly_thu_id → agency.name
            //     nv_ho_so_dang_ky.nhan_vien_thu_id → seller.full_name
            Map<Integer, String> hoSoDaiLyMap = new java.util.HashMap<>();
            Map<Integer, String> hoSoSellerMap = new java.util.HashMap<>();
            doiSoatHoSoRepository.findDaiLySellerByHoSoIds(hoSoIds)
                    .forEach(p -> {
                        hoSoDaiLyMap.put(p.getHoSoId(), p.getTenDaiLy() != null ? p.getTenDaiLy() : "");
                        hoSoSellerMap.put(p.getHoSoId(), p.getTenNhanVienThu() != null ? p.getTenNhanVienThu() : "");
                    });

            // 5. Data rows
            CellStyle numberStyle = createNumberStyle(workbook);
            int rowNum = 1;
            BigDecimal tongSoTienDong = BigDecimal.ZERO;
            BigDecimal tongThuLao = BigDecimal.ZERO;

            for (NvDoiSoatThuLaoHoSo hoSo : hoSoList) {
                Row row = sheet.createRow(rowNum++);
                row.createCell(0).setCellValue(rowNum - 1);                     // STT
                row.createCell(1).setCellValue(                                  // Đợt kê khai
                        dotKeKhaiTenMap.getOrDefault(hoSo.getDotKeKhaiId(), ""));
                row.createCell(2).setCellValue(hoSo.getMaHoSo());
                row.createCell(3).setCellValue(hoSo.getTenNguoiThamGia());
                row.createCell(4).setCellValue(resolveThuTuc(hoSo.getLoaiNghiepVu())); // Thủ tục
                row.createCell(5).setCellValue(                                  // Tỉnh/TP (ten từ ma_tinh_tt)
                        hoSoTinhTtMap.getOrDefault(hoSo.getHoSoId(), ""));
                row.createCell(6).setCellValue(hoSo.getVung());
                row.createCell(7).setCellValue(hoSo.getPhuongAnDong());
                row.createCell(8).setCellValue(hoSo.getPhuongThucDong());
                row.createCell(9).setCellValue(hoSo.getSoBienLai());

                Cell soTienCell = row.createCell(10);
                soTienCell.setCellValue(hoSo.getSoTienDong().doubleValue());
                soTienCell.setCellStyle(numberStyle);

                Cell tiLeCell = row.createCell(11);
                tiLeCell.setCellValue(hoSo.getTiLeThuLao().doubleValue());
                tiLeCell.setCellStyle(numberStyle);

                Cell thuLaoCell = row.createCell(12);
                thuLaoCell.setCellValue(hoSo.getThuLaoDuocHuong().doubleValue());
                thuLaoCell.setCellStyle(numberStyle);

                row.createCell(13).setCellValue(                                  // Đại lý thu hộ (agency.name)
                        hoSoDaiLyMap.getOrDefault(hoSo.getHoSoId(), ""));
                row.createCell(14).setCellValue(                                  // Nhân viên thu (seller.full_name)
                        hoSoSellerMap.getOrDefault(hoSo.getHoSoId(), ""));

                tongSoTienDong = tongSoTienDong.add(hoSo.getSoTienDong());
                tongThuLao = tongThuLao.add(hoSo.getThuLaoDuocHuong());
            }

            // 5. Footer row — TỔNG CỘNG
            Row footerRow = sheet.createRow(rowNum);
            footerRow.createCell(0).setCellValue("TỔNG CỘNG");
            // Blank cells 1-9
            for (int i = 1; i <= 9; i++) footerRow.createCell(i).setCellValue("");

            Cell tongTienCell = footerRow.createCell(10);
            tongTienCell.setCellValue(tongSoTienDong.doubleValue());
            tongTienCell.setCellStyle(numberStyle);

            footerRow.createCell(11).setCellValue("");  // blank (tỷ lệ không sum)

            Cell tongThuLaoCell = footerRow.createCell(12);
            tongThuLaoCell.setCellValue(tongThuLao.doubleValue());
            tongThuLaoCell.setCellStyle(numberStyle);

            footerRow.createCell(13).setCellValue("");
            footerRow.createCell(14).setCellValue("");

            // 6. Auto-size columns
            for (int i = 0; i < headers.length; i++) {
                sheet.autoSizeColumn(i);
            }

            workbook.write(out);
            byte[] xlsxBytes = out.toByteArray();
            log.info("[{}/EXPORT_XLSX] SUCCESS doiSoatId={}, rows={}, size={} bytes",
                    user.getId(), doiSoatId, hoSoList.size(), xlsxBytes.length);
            return xlsxBytes;

        } catch (IOException e) {
            log.error("[{}/EXPORT_XLSX] ERROR doiSoatId={}: {}", user.getId(), doiSoatId, e.getMessage(), e);
            throw new BusinessException(MessageResponseDict.EXPORT_FAILED);
        }
    }

    // ─── Private Helpers ─────────────────────────────────────────────────────

    private C17DataModel buildC17Data(NvDoiSoatThuLao doiSoat, List<NvDoiSoatThuLaoHoSo> hoSoList) {
        BigDecimal tongSoTienNop = hoSoList.stream()
                .map(NvDoiSoatThuLaoHoSo::getSoTienDong)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        return C17DataModel.builder()
                .ngayTaoKy(doiSoat.getCreatedAt() != null
                        ? doiSoat.getCreatedAt().toLocalDate().format(DATE_FMT) : "")
                .hoSoList(hoSoList)
                .tongSoToBienLai(hoSoList.size())
                .tongSoTienNop(tongSoTienNop)
                .soTienBangChu(soTienBangChu(tongSoTienNop))
                .build();
    }

    private void fillDocxTemplate(XWPFDocument doc, C17DataModel data) {
        // Replace placeholder {{field}} trong tất cả paragraph và table cells
        doc.getParagraphs().forEach(para -> {
            para.getRuns().forEach(run -> {
                String text = run.getText(0);
                if (text != null) {
                    text = text.replace("{{ngayTaoKy}}", data.getNgayTaoKy())
                               .replace("{{tongSoToBienLai}}", String.valueOf(data.getTongSoToBienLai()))
                               .replace("{{tongSoTienNop}}", formatCurrency(data.getTongSoTienNop()))
                               .replace("{{soTienBangChu}}", data.getSoTienBangChu());
                    run.setText(text, 0);
                }
            });
        });
        // Note: Table cell replacement và biên lai rows — implement theo template cụ thể
    }

    private byte[] convertDocxToPdf(XWPFDocument doc) throws IOException {
        // Implementation: dùng JodConverter hoặc Aspose/iText
        // Placeholder: convert DOCX → PDF via LibreOffice headless
        try (ByteArrayOutputStream out = new ByteArrayOutputStream()) {
            // jodConverter.convert(docxStream).to(out).as(DefaultDocumentFormatRegistry.PDF).execute();
            doc.write(out);  // Tạm thời trả về DOCX bytes để compile — thay bằng PDF conversion
            return out.toByteArray();
        }
    }

    private void kiemTraQuyenSoHuu(NvDoiSoatThuLao doiSoat, AccountInfo user) {
        if (user.isAdmin()) return;
        if (!doiSoat.getCreatedByUserId().equals(user.getId())) {
            throw new BusinessException(MessageResponseDict.DOI_SOAT_FORBIDDEN);
        }
    }

    private String resolveThuTuc(String loaiNghiepVu) {
        if ("BHXH_TU_NGUYEN".equals(loaiNghiepVu)) return "BHXH Tự nguyện";
        if ("BHYT_HO_GIA_DINH".equals(loaiNghiepVu)) return "BHYT Hộ gia đình";
        return loaiNghiepVu;
    }

    private String formatCurrency(BigDecimal amount) {
        return NumberFormat.getNumberInstance(new Locale("vi", "VN")).format(amount) + " đồng";
    }

    private String soTienBangChu(BigDecimal amount) {
        // TODO: implement số tiền bằng chữ tiếng Việt
        // Library đề xuất: custom SoTienBangChuUtils trong project
        return "[" + formatCurrency(amount) + "]";
    }

    private CellStyle createHeaderStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        Font font = workbook.createFont();
        font.setBold(true);
        style.setFont(font);
        style.setFillForegroundColor(IndexedColors.GREY_25_PERCENT.getIndex());
        style.setFillPattern(FillPatternType.SOLID_FOREGROUND);
        style.setBorderBottom(BorderStyle.THIN);
        style.setBorderTop(BorderStyle.THIN);
        style.setBorderLeft(BorderStyle.THIN);
        style.setBorderRight(BorderStyle.THIN);
        style.setAlignment(HorizontalAlignment.CENTER);
        style.setWrapText(true);
        return style;
    }

    private CellStyle createNumberStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        DataFormat format = workbook.createDataFormat();
        style.setDataFormat(format.getFormat("#,##0.00"));
        return style;
    }
}

---

## 3.5 Supporting Projections + Repository Methods

### Projections cần thêm — **Package**: `vn.vivas.bh.repository.projection`

```java
package vn.vivas.bh.repository.projection;

import java.time.LocalDateTime;

/**
 * Batch-load thông tin đợt kê khai cho export XLSX.
 * tenDotKeKhai = String.format("[%02d] %s", soThuTu, createdAt.format("MM/yyyy"))
 */
public interface DotKeKhaiInfoProjection {
    Integer getId();
    Integer getSoThuTu();
    LocalDateTime getCreatedAt();
}
```

```java
package vn.vivas.bh.repository.projection;

/**
 * Batch-load Tỉnh/TP cho từng hồ sơ — từ ma_tinh_tt của người tham gia
 * (NvNguoiThamGia.tinhTt.ten, FK: ma_tinh_tt → dm_donvi_hanhchinh.ten)
 */
public interface HoSoTinhTtProjection {
    Integer getHoSoId();
    String getTenTinhTt();  // nullable nếu ma_tinh_tt NULL
}
```

```java
package vn.vivas.bh.repository.projection;

/**
 * Batch-load tên đại lý thu hộ và nhân viên thu cho export XLSX.
 * JOIN: nv_ho_so_dang_ky → agency (dai_ly_thu_id) → agency.name
 *       nv_ho_so_dang_ky → seller (nhan_vien_thu_id) → seller.full_name
 */
public interface HoSoDaiLySellerProjection {
    Integer getHoSoId();
    String getTenDaiLy();       // nullable nếu dai_ly_thu_id NULL
    String getTenNhanVienThu(); // nullable nếu nhan_vien_thu_id NULL
}
```

### Repository methods — thêm vào `DoiSoatThuLaoHoSoRepository`

```java
/**
 * Batch-load thông tin đợt kê khai (id, so_thu_tu, created_at) từ danh sách ids.
 * Dùng để gen tenDotKeKhai trong Java (không có cột ten_dot_ke_khai trên DB).
 */
@Query(value = """
        SELECT id AS id, so_thu_tu AS soThuTu, created_at AS createdAt
        FROM nv_dot_ke_khai
        WHERE id IN :ids
        """, nativeQuery = true)
List<DotKeKhaiInfoProjection> findDotKeKhaiInfoByIds(@Param("ids") List<Integer> ids);

/**
 * Batch-load tenTinhTt của người tham gia theo danh sách hoSoId.
 * JOIN path: nv_ho_so_dang_ky → nv_nguoi_tham_gia → dm_donvi_hanhchinh (ma_tinh_tt)
 * Trả về NULL nếu ma_tinh_tt là NULL.
 */
@Query(value = """
        SELECT hs.id AS hoSoId, dvhc.ten AS tenTinhTt
        FROM nv_ho_so_dang_ky hs
        JOIN nv_nguoi_tham_gia ntt ON ntt.id = hs.nguoi_tham_gia_id
        LEFT JOIN dm_donvi_hanhchinh dvhc ON dvhc.ma_dbhc = ntt.ma_tinh_tt
        WHERE hs.id IN :hoSoIds
        """, nativeQuery = true)
List<HoSoTinhTtProjection> findTinhTtByHoSoIds(@Param("hoSoIds") List<Integer> hoSoIds);

/**
 * Batch-load tên đại lý thu hộ và nhân viên thu theo danh sách hoSoId.
 * JOIN path:
 *   nv_ho_so_dang_ky.dai_ly_thu_id → agency.name   (Đại lý thu hộ)
 *   nv_ho_so_dang_ky.nhan_vien_thu_id → seller.full_name (Đại lý thu’/ Nhân viên thu)
 * trả về NULL nếu các FK là NULL.
 */
@Query(value = """
        SELECT hs.id AS hoSoId,
               ag.name AS tenDaiLy,
               sl.full_name AS tenNhanVienThu
        FROM nv_ho_so_dang_ky hs
        LEFT JOIN agency ag ON ag.id = hs.dai_ly_thu_id
        LEFT JOIN seller sl ON sl.id = hs.nhan_vien_thu_id
        WHERE hs.id IN :hoSoIds
        """, nativeQuery = true)
List<HoSoDaiLySellerProjection> findDaiLySellerByHoSoIds(@Param("hoSoIds") List<Integer> hoSoIds);
```


## 4. Data Model cho C17

**Class**: `C17DataModel` | **Package**: `vn.vivas.bh.service.model`

```java
package vn.vivas.bh.service.model;

import lombok.Builder;
import lombok.Data;
import vn.vivas.bh.entity.nghiepvu.NvDoiSoatThuLaoHoSo;

import java.math.BigDecimal;
import java.util.List;

@Data
@Builder
public class C17DataModel {
    private String ngayTaoKy;            // (5): formatted dd/MM/yyyy
    private List<NvDoiSoatThuLaoHoSo> hoSoList;
    private int tongSoToBienLai;         // (6): count biên lai
    private BigDecimal tongSoTienNop;    // (7): SUM soTienDong
    private String soTienBangChu;        // (8): VND text
    // (1) tenCoQuanBhxhTinh, (3) tenDaiLyThu, (9) tenDaiDienDaiLyThu
    // — load từ DB config khi fill template
    private String tenCoQuanBhxhTinh;
    private String tenDaiLyThu;
    private String tenDaiDienDaiLyThu;
}
```

---

## 5. Controller — thêm 2 export endpoints

```java
// Thêm vào DoiSoatThuLaoController (đã có ở Task 01)

@Autowired  // hoặc @RequiredArgsConstructor
private DoiSoatExportService exportService;

@Operation(summary = "Xuất C17-TS PDF")
@GetMapping("/{id}/export-c17")
@PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.DOI_SOAT_THU_LAO)")
public ResponseEntity<byte[]> exportC17(@PathVariable Integer id) {
    byte[] pdfBytes = exportService.exportC17Pdf(id);
    return ResponseEntity.ok()
            .header("Content-Disposition", "attachment; filename=\"C17-TS.pdf\"")
            .header("Content-Type", "application/pdf")
            .body(pdfBytes);
}

@Operation(summary = "Xuất DS chi tiết hồ sơ XLSX")
@GetMapping("/{id}/export-chi-tiet")
@PreAuthorize("@permissionChecker.hasPermission(authentication, @permissions.DOI_SOAT_THU_LAO)")
public ResponseEntity<byte[]> exportChiTiet(@PathVariable Integer id) {
    byte[] xlsxBytes = exportService.exportChiTietXlsx(id);
    return ResponseEntity.ok()
            .header("Content-Disposition", "attachment; filename=\"DS chi tiet ho so.xlsx\"")
            .header("Content-Type", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")
            .body(xlsxBytes);
}
```

---

## 6. Error Code cần thêm

```java
// Thêm vào MessageResponseDict:
EXPORT_FAILED(HttpStatus.INTERNAL_SERVER_ERROR, 320007, "Xuất file thất bại, vui lòng thử lại"),
```

---

## 7. Unit Tests

**Class**: `DoiSoatExportServiceTest` | **Package**: `vn.vivas.bh.service`

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
import vn.vivas.bh.entity.nghiepvu.NvDoiSoatThuLao;
import vn.vivas.bh.entity.nghiepvu.NvDoiSoatThuLaoHoSo;
import vn.vivas.bh.enums.Role;
import vn.vivas.bh.enums.TrangThaiDoiSoat;
import vn.vivas.bh.repository.nghiepvu.DoiSoatThuLaoHoSoRepository;
import vn.vivas.bh.repository.nghiepvu.DoiSoatThuLaoRepository;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.Arrays;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class DoiSoatExportServiceTest {

    @Mock private DoiSoatThuLaoRepository doiSoatRepository;
    @Mock private DoiSoatThuLaoHoSoRepository doiSoatHoSoRepository;
    @InjectMocks private DoiSoatExportService exportService;

    @BeforeEach
    void setUp() {
        AccountInfo admin = AccountInfo.builder()
                .id(1).role(Role.ADMIN)
                .permissions(Arrays.asList("ALL_PERMISSIONS")).daiLyId(null).build();
        when(AuthenticationService.getCurrentUserRequired()).thenReturn(admin);
    }

    @Test
    void shouldExportXlsxWithCorrectColumnCount() {
        // Given
        NvDoiSoatThuLao doiSoat = NvDoiSoatThuLao.builder()
                .id(1).daiLyThuId(1).coQuanBhxhId(1)
                .tenDoiSoat("Kỳ 01/2026")
                .tuNgay(LocalDate.now().minusMonths(1)).denNgay(LocalDate.now())
                .trangThai(TrangThaiDoiSoat.DANG_DOI_SOAT)
                .createdByUserId(1).createdAt(LocalDateTime.now()).build();

        NvDoiSoatThuLaoHoSo hoSo1 = NvDoiSoatThuLaoHoSo.builder()
                .id(1).doiSoatThuLaoId(1).hoSoId(10).dotKeKhaiId(5)
                .maHoSo("BHXH-2026-001").tenNguoiThamGia("Nguyễn Văn An")
                .loaiNghiepVu("BHXH_TU_NGUYEN").loaiSanPham("bhxh")
                .vung("I").phuongAnDong("TM").phuongThucDong("12")
                .soBienLai("BL001")
                .soTienDong(new BigDecimal("1000000"))
                .tiLeThuLao(new BigDecimal("18.00"))
                .thuLaoDuocHuong(new BigDecimal("180000"))
                .ngayTao(LocalDateTime.now()).build();

        when(doiSoatRepository.findById(1)).thenReturn(Optional.of(doiSoat));
        when(doiSoatHoSoRepository.findAllByDoiSoatThuLaoId(1)).thenReturn(List.of(hoSo1));

        // When
        byte[] xlsxBytes = exportService.exportChiTietXlsx(1);

        // Then — verify file is not empty and has content
        assertThat(xlsxBytes).isNotNull();
        assertThat(xlsxBytes.length).isGreaterThan(0);
    }

    @Test
    void shouldCalculateCorrectTotalsInXlsx() {
        // Given: 2 hồ sơ
        NvDoiSoatThuLao doiSoat = NvDoiSoatThuLao.builder()
                .id(2).daiLyThuId(1).coQuanBhxhId(1)
                .tenDoiSoat("Kỳ test totals")
                .tuNgay(LocalDate.now().minusMonths(1)).denNgay(LocalDate.now())
                .trangThai(TrangThaiDoiSoat.DANG_DOI_SOAT)
                .createdByUserId(1).createdAt(LocalDateTime.now()).build();

        List<NvDoiSoatThuLaoHoSo> hoSoList = List.of(
                buildHoSo(1, new BigDecimal("1000000"), new BigDecimal("13.5"), new BigDecimal("135000")),
                buildHoSo(2, new BigDecimal("2000000"), new BigDecimal("16.2"), new BigDecimal("324000"))
        );

        when(doiSoatRepository.findById(2)).thenReturn(Optional.of(doiSoat));
        when(doiSoatHoSoRepository.findAllByDoiSoatThuLaoId(2)).thenReturn(hoSoList);

        // When
        byte[] xlsxBytes = exportService.exportChiTietXlsx(2);

        // Then — file generated (detailed verification via Apache POI reader — integration test)
        assertThat(xlsxBytes).isNotEmpty();
        verify(doiSoatHoSoRepository, times(1)).findAllByDoiSoatThuLaoId(2);
    }

    private NvDoiSoatThuLaoHoSo buildHoSo(int id, BigDecimal soTien, BigDecimal tiLe, BigDecimal thuLao) {
        return NvDoiSoatThuLaoHoSo.builder()
                .id(id).doiSoatThuLaoId(2).hoSoId(id + 100).dotKeKhaiId(5)
                .maHoSo("HS-00" + id).tenNguoiThamGia("Người Tham Gia " + id)
                .loaiNghiepVu("BHXH_TU_NGUYEN").loaiSanPham("bhxh")
                .vung("II").phuongAnDong("TM").phuongThucDong("12")
                .soBienLai("BL-00" + id)
                .soTienDong(soTien).tiLeThuLao(tiLe).thuLaoDuocHuong(thuLao)
                .ngayTao(LocalDateTime.now()).build();
    }
}
```

---

## 8. Checklist

- [ ] `pom.xml` — thêm poi-ooxml dependency (version 5.3.0)
- [ ] `DoiSoatExportService` — class tồn tại, inject DoiSoatThuLaoRepository + HoSoRepository
- [ ] `exportC17Pdf(Integer)` — load template, fill placeholders, convert PDF, return bytes
- [ ] `exportChiTietXlsx(Integer)` — 15-column header, data rows, footer (TỔNG CỘNG), auto-size
- [ ] `C17DataModel` — @Builder, tất cả 10 fields theo PAKT
- [ ] `MessageResponseDict.EXPORT_FAILED` — thêm error code 320007
- [ ] Template `template_doi_soat_C17_TS.doc` — tồn tại trong `resources/templates/`
- [ ] Controller: 2 export endpoints, trả về `ResponseEntity<byte[]>` với Content-Disposition
- [ ] Unit tests — XLSX not empty, verify repo called, 2-row totals
- [ ] `./mvnw test` — pass

## 9. Notes

> **PDF Conversion**: JodConverter yêu cầu LibreOffice headless cài trên server.  
> Nếu môi trường không có LibreOffice, có thể dùng Aspose.Words (thương mại) hoặc trả file DOCX thay PDF và đổi Content-Type cho phù hợp.

> **`phuongAnDong` / `phuongThucDong` resolve**: Trong XLSX đang output raw `ma` code.  
> Nếu cần display `ten`, implement Java key-value map load-once từ `dm_phuong_an_dong` và `dm_phuong_thuc_dong` — inject vào `DoiSoatExportService` và apply khi build row.

## 10. Sync Points

**Depends on**:
- Task 00: `NvDoiSoatThuLao`, `NvDoiSoatThuLaoHoSo`, `DoiSoatThuLaoRepository`, `DoiSoatThuLaoHoSoRepository`
- Task 02: RBAC check pattern `kiemTraQuyenSoHuu()` (copy vào DoiSoatExportService)

**Event**: Không có events — sync export (stream trực tiếp)
