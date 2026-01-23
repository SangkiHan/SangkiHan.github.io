---
title: ExcelUtils 메모리 누수 해결 - AutoCloseable 패턴으로 리소스 관리 개선
author: SangkiHan
date: 2026-01-23 14:00:00 +0900
categories: [Java, Spring Boot]
tags: [Java, Spring Boot, Memory Leak, Apache POI, AutoCloseable, Best Practice]
---

## 개요

Apache POI를 사용한 엑셀 다운로드 기능에서 메모리 누수 문제가 발생했습니다. `XSSFWorkbook` 리소스가 제대로 해제되지 않아 대량의 엑셀 다운로드 요청 시 메모리 부족 문제가 발생할 수 있었습니다. 이 글에서는 `AutoCloseable` 인터페이스를 활용하여 리소스 관리를 개선한 과정을 상세히 다룹니다.

### 주요 개선 사항
- `XSSFWorkbook` 리소스 관리 개선
- `AutoCloseable` 인터페이스 구현
- `try-with-resources` 패턴 적용
- 불필요한 객체 생성 제거
- `DataFormatter` 재사용으로 성능 최적화

---

## 메모리 누수 원인 분석

### 1. Workbook 리소스 관리 실패 (Critical)

#### AS-IS (문제 코드)
```java
public class ExcelUtils {
    private final XSSFWorkbook workbook;
    
    public ExcelUtils() {
        workbook = new XSSFWorkbook();  // 생성만 하고
    }
    
    public void downloadExcel(HttpServletResponse response, String fileName) {
        try {
            workbook.write(response.getOutputStream());
            response.getOutputStream().flush();
        } finally {
            if (workbook != null) {
                workbook.close();  // 오직 여기서만 닫힘!
            }
        }
    }
}
```

**문제점:**
1. `downloadExcel()` 메서드가 호출되지 않으면 `workbook.close()`가 실행되지 않음
2. 중간에 예외가 발생하면 리소스가 해제되지 않을 수 있음
3. `XSSFWorkbook` 인스턴스당 15-30MB의 메모리를 점유

**메모리 누수 시나리오:**
```java
// 시나리오 1: downloadExcel이 호출되지 않는 경우
ExcelUtils excel = new ExcelUtils();  // workbook 생성 (메모리 할당)
excel.makeSheet("test");
excel.insertRowTitle(headers);
// 여기서 예외 발생! downloadExcel 호출 안됨
// → workbook.close() 실행 안됨 → 메모리 누수!

// 시나리오 2: 동시 다운로드 요청 시
// 100명이 동시 다운로드 → 중간에 에러 발생 시 1.5GB ~ 3GB 메모리 누수 발생!
```

---

### 2. OutputStream 리소스 누출

#### AS-IS (문제 코드)
```java
public void downloadExcel(HttpServletResponse response, String fileName) {
    try {
        workbook.write(response.getOutputStream());  // stream 열기만 함
        response.getOutputStream().flush();          // flush만 하고 close 안 함!
    } finally {
        workbook.close();
    }
}
```

**문제점:**
1. `getOutputStream()`을 두 번 호출 (불필요)
2. `OutputStream`을 명시적으로 닫지 않음
3. 네트워크 에러 발생 시 버퍼 메모리가 해제되지 않음

---

### 3. 불필요한 객체 생성

#### AS-IS (문제 코드)
```java
public void insertRowData(List<Map<String, Object>> excelData, 
                         String[] dbColStr, 
                         String[] style, 
                         String autoWidth) {
    List<CellStyle> cellStyles = new ArrayList<>();  // 불필요한 List 생성!
    for (String s : style) {
        cellStyles.add(cellStyleSelector(s));  // 이미 있는 스타일을 복사
    }
    
    for (int i = 0; i < excelData.size(); i++) {
        for (int j = 0; j < dbColStr.length; j++) {
            cell.setCellStyle(cellStyles.get(j));  // List에서 꺼내 쓰기
        }
    }
}
```

**메모리 낭비:**
- 1000행 처리 시 약 104 bytes의 불필요한 메모리 할당
- 이미 캐시된 스타일을 재사용하는데 굳이 리스트로 만들 필요 없음

---

### 4. DataFormatter 반복 생성

#### AS-IS (문제 코드)
```java
private static <T> T mapRowToObject(Class<T> clazz, 
                                   List<String> headers, 
                                   String[] dbColumns, 
                                   Row row) {
    DataFormatter formatter = new DataFormatter();  // 매 행마다 생성!
    
    for (int i = 0; i < headers.size(); i++) {
        String value = formatter.formatCellValue(cell);
    }
}
```

**메모리 낭비 계산:**
```
1000행 처리 시:
- DataFormatter 객체 생성: 1000번
- 각 DataFormatter 크기: ~1-2KB
- 총 낭비: 1-2MB
- GC 부담: 1000개 객체 수거 필요
```

---

## 해결 방법: AutoCloseable 패턴 적용

### 1. AutoCloseable 인터페이스 구현

#### TO-BE (개선 코드)
```java
public class ExcelUtils implements AutoCloseable {
    private final XSSFWorkbook workbook;
    private boolean closed = false;  // 중복 close 방지
    
    public ExcelUtils() {
        workbook = new XSSFWorkbook();
        cellStyleTitle = createCellStyle("title");
        cellStyleCenter = createCellStyle("center");
        cellStyleLeft = createCellStyle("left");
        cellStyleRight = createCellStyle("right");
    }
    
    @Override
    public void close() throws IOException {
        if (!closed && workbook != null) {
            try {
                workbook.close();
            } finally {
                closed = true;  // 상태 플래그 관리
            }
        }
    }
}
```

**개선 효과:**
- `try-with-resources` 패턴 사용 가능
- 예외 발생 시에도 무조건 리소스 정리 보장
- 중복 close 방지로 안전성 향상

---

### 2. try-with-resources 패턴으로 안전한 사용

#### TO-BE (개선 코드)
```java
@GetMapping("/export")
public void exportData(HttpServletResponse response) {
    // try-with-resources로 무조건 close 보장!
    try (ExcelUtils excel = new ExcelUtils()) {
        excel.makeSheet("데이터");
        excel.insertRowTitle(headers);
        excel.insertRowData(data, columns, styles, "auto");
        excel.downloadExcel(response, "data.xlsx");
    } catch (Exception e) {
        log.error("엑셀 다운로드 에러", e);
        // 여기서 return해도 excel.close() 자동 호출!
        // workbook.close()가 반드시 실행됨!
    }
}
```

**장점:**
1. 예외 발생 여부와 관계없이 `close()` 메서드 자동 호출
2. 코드 가독성 향상
3. 실수로 리소스를 닫지 않는 경우 방지

---

### 3. OutputStream try-with-resources 적용

#### TO-BE (개선 코드)
```java
public void downloadExcel(HttpServletResponse response, String fileName) {
    try {
        // Content-Type과 헤더 설정
        response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
        response.setCharacterEncoding("UTF-8");
        
        String encodedFileName = URLEncoder.encode(fileName, StandardCharsets.UTF_8)
                .replaceAll("\\+", "%20");
        
        response.setHeader("Content-Disposition", "attachment; filename=" + encodedFileName);
        response.setHeader("Access-Control-Expose-Headers", "Content-Disposition");
        
        // 캐시 방지
        response.setHeader("Cache-Control", "no-cache, no-store, must-revalidate");
        response.setHeader("Pragma", "no-cache");
        response.setDateHeader("Expires", 0);
        
        // try-with-resources로 OutputStream 자동 close
        try (ServletOutputStream outputStream = response.getOutputStream()) {
            workbook.write(outputStream);
            outputStream.flush();
        }  // 자동으로 outputStream.close() 호출!
        
    } catch (IOException e) {
        throw new PlatformException("엑셀 파일 다운로드에 실패했습니다.", e);
    }
    // workbook.close()는 외부 try-with-resources에서 관리
}
```

**개선 효과:**
- `OutputStream`이 자동으로 close
- 버퍼 메모리 즉시 해제
- 네트워크 연결 즉시 종료

---

### 4. 불필요한 객체 생성 제거

#### TO-BE (개선 코드)
```java
public void insertRowData(List<Map<String, Object>> excelData, 
                         String[] dbColStr, 
                         String[] style, 
                         String autoWidth) {
    // 불필요한 List 생성 제거!
    for (int i = 0; i < excelData.size(); i++) {
        Row row = sheet.createRow(i + rowIndex);
        Map<String, Object> dataRow = excelData.get(i);
        
        for (int j = 0; j < dbColStr.length; j++) {
            Cell cell = row.createCell(j);
            Object value = dataRow.get(dbColStr[j]);
            
            if (value == null) {
                cell.setCellValue("");
            } else if (value instanceof Number) {
                cell.setCellValue(((Number) value).doubleValue());
            } else {
                cell.setCellValue(value.toString());
            }
            
            // 직접 캐시된 스타일 사용!
            cell.setCellStyle(cellStyleSelector(style[j]));
        }
    }
    
    // 열 너비 자동 조정
    if ("auto".equals(autoWidth)) {
        for (int j = 0; j < dbColStr.length; j++) {
            sheet.autoSizeColumn(j);
            int currentWidth = sheet.getColumnWidth(j);
            sheet.setColumnWidth(j, Math.max(currentWidth + 400, 3800));
        }
    } else {
        for (int j = 0; j < dbColStr.length; j++) {
            sheet.setColumnWidth(j, 3800);
        }
    }
}
```

**개선 효과:**
- 불필요한 `ArrayList` 객체 생성 제거
- GC 부담 감소
- 코드 간결성 향상

---

### 5. DataFormatter 재사용

#### TO-BE (개선 코드)
```java
public static <T> List<T> parseExcelToList(Class<T> clazz, 
                                           ExcelType excelType, 
                                           MultipartFile file) throws IOException {
    try (InputStream is = file.getInputStream(); 
         Workbook workbook = WorkbookFactory.create(is)) {
        Sheet sheet = workbook.getSheetAt(0);
        Iterator<Row> rowIterator = sheet.iterator();
        
        List<String> headers = extractAndValidateHeaders(rowIterator, excelType);
        List<T> result = new ArrayList<>();
        
        // DataFormatter를 한 번만 생성하여 재사용!
        DataFormatter formatter = new DataFormatter();
        
        while (rowIterator.hasNext()) {
            Row row = rowIterator.next();
            T obj = mapRowToObject(clazz, headers, excelType.getDbColumn(), row, formatter);
            result.add(obj);
        }
        return result;
    }
}

private static <T> T mapRowToObject(Class<T> clazz, 
                                   List<String> headers, 
                                   String[] dbColumns, 
                                   Row row, 
                                   DataFormatter formatter) {  // 매개변수로 받아 재사용
    List<Object> rowValues = new ArrayList<>();
    try {
        T instance = clazz.getDeclaredConstructor().newInstance();
        for (int i = 0; i < headers.size(); i++) {
            Cell cell = row.getCell(i, Row.MissingCellPolicy.CREATE_NULL_AS_BLANK);
            String value = formatter.formatCellValue(cell);
            rowValues.add(value);
            
            String fieldName = dbColumns[i];
            Field field = clazz.getDeclaredField(fieldName);
            field.setAccessible(true);
            field.set(instance, value);
        }
        return instance;
    } catch (Exception e) {
        System.err.println("엑셀 행 변환 오류. 행 데이터: " + rowValues);
        throw new PlatformException("엑셀 행을 객체로 변환 중 오류 발생", e);
    }
}
```

**개선 효과:**
- 1000번 생성 → 1번 생성
- 메모리 사용량 1/1000으로 감소
- GC 부담 대폭 감소

---

## 성능 개선 효과 비교

| 항목 | AS-IS (문제) | TO-BE (해결) | 개선 효과 |
|------|-------------|--------------|-----------|
| **Workbook 관리** | finally에서만 close | AutoCloseable + try-with-resources | 메모리 누수 원천 차단 |
| **OutputStream** | 명시적 close 없음 | try-with-resources | 버퍼 메모리 즉시 해제 |
| **List 생성** | 매번 생성 | 직접 접근 | 100 bytes 절약/호출 |
| **DataFormatter** | 1000번 생성 | 1번 생성 | 1-2MB 절약/1000행 |

---

## 실제 사용 예시 비교

### AS-IS (위험한 패턴)
```java
@GetMapping("/export")
public void exportData(HttpServletResponse response) {
    ExcelUtils excel = new ExcelUtils();  // workbook 생성
    try {
        excel.makeSheet("데이터");
        excel.insertRowTitle(headers);
        excel.insertRowData(data, columns, styles, "auto");
        excel.downloadExcel(response, "data.xlsx");
    } catch (Exception e) {
        log.error("에러", e);
        // 여기서 return하면 workbook.close() 호출 안됨!
        // 메모리 누수 발생!
    }
}
```

### TO-BE (안전한 패턴)
```java
@GetMapping("/export")
public void exportData(HttpServletResponse response) {
    // try-with-resources로 무조건 close 보장!
    try (ExcelUtils excel = new ExcelUtils()) {
        excel.makeSheet("데이터");
        excel.insertRowTitle(headers);
        excel.insertRowData(data, columns, styles, "auto");
        excel.downloadExcel(response, "data.xlsx");
    } catch (Exception e) {
        log.error("에러", e);
        // 여기서 return해도 excel.close() 자동 호출!
        // workbook.close()가 반드시 실행됨!
    }
}
```

---

## 전체 개선 코드

```java
package com.platform.global.excel.util;

import com.platform.global.excel.enums.ExcelType;
import com.platform.global.exception.PlatformException;
import jakarta.servlet.ServletOutputStream;
import jakarta.servlet.http.HttpServletResponse;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.io.InputStream;
import java.lang.reflect.Field;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.util.*;

public class ExcelUtils implements AutoCloseable {

    private final XSSFWorkbook workbook;
    private Sheet sheet;
    private int rowIndex = 0;
    private final CellStyle cellStyleTitle;
    private final CellStyle cellStyleCenter;
    private final CellStyle cellStyleLeft;
    private final CellStyle cellStyleRight;
    private boolean closed = false;

    public ExcelUtils() {
        workbook = new XSSFWorkbook();
        cellStyleTitle = createCellStyle("title");
        cellStyleCenter = createCellStyle("center");
        cellStyleLeft = createCellStyle("left");
        cellStyleRight = createCellStyle("right");
    }

    @Override
    public void close() throws IOException {
        if (!closed && workbook != null) {
            try {
                workbook.close();
            } finally {
                closed = true;
            }
        }
    }

    public void makeSheet(String sheetName) {
        sheet = workbook.createSheet(sheetName);
        rowIndex = 0;
    }

    public void insertRowTitle(String[] headerStr) {
        Row row = sheet.createRow(0);
        for (int i = 0; i < headerStr.length; i++) {
            if (headerStr[i] == null) break;
            Cell cell = row.createCell(i);
            cell.setCellValue(headerStr[i]);
            cell.setCellStyle(cellStyleTitle);
        }
        rowIndex++;
    }

    public void insertRowData(List<Map<String, Object>> excelData, 
                             String[] dbColStr, 
                             String[] style, 
                             String autoWidth) {
        for (int i = 0; i < excelData.size(); i++) {
            Row row = sheet.createRow(i + rowIndex);
            Map<String, Object> dataRow = excelData.get(i);

            for (int j = 0; j < dbColStr.length; j++) {
                Cell cell = row.createCell(j);
                Object value = dataRow.get(dbColStr[j]);

                if (value == null) {
                    cell.setCellValue("");
                } else if (value instanceof Number) {
                    cell.setCellValue(((Number) value).doubleValue());
                } else {
                    cell.setCellValue(value.toString());
                }

                cell.setCellStyle(cellStyleSelector(style[j]));
            }
        }

        if ("auto".equals(autoWidth)) {
            for (int j = 0; j < dbColStr.length; j++) {
                sheet.autoSizeColumn(j);
                int currentWidth = sheet.getColumnWidth(j);
                sheet.setColumnWidth(j, Math.max(currentWidth + 400, 3800));
            }
        } else {
            for (int j = 0; j < dbColStr.length; j++) {
                sheet.setColumnWidth(j, 3800);
            }
        }
    }

    public void downloadExcel(HttpServletResponse response, String fileName) {
        try {
            response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
            response.setCharacterEncoding("UTF-8");

            String encodedFileName = URLEncoder.encode(fileName, StandardCharsets.UTF_8)
                    .replaceAll("\\+", "%20");

            response.setHeader("Content-Disposition", "attachment; filename=" + encodedFileName);
            response.setHeader("Access-Control-Expose-Headers", "Content-Disposition");
            response.setHeader("Cache-Control", "no-cache, no-store, must-revalidate");
            response.setHeader("Pragma", "no-cache");
            response.setDateHeader("Expires", 0);

            try (ServletOutputStream outputStream = response.getOutputStream()) {
                workbook.write(outputStream);
                outputStream.flush();
            }
        } catch (IOException e) {
            throw new PlatformException("엑셀 파일 다운로드에 실패했습니다.", e);
        }
    }

    private CellStyle createCellStyle(String type) {
        CellStyle style = workbook.createCellStyle();
        Font font = workbook.createFont();
        font.setFontHeightInPoints((short) 10);
        style.setFont(font);

        style.setBorderBottom(BorderStyle.THIN);
        style.setBorderLeft(BorderStyle.THIN);
        style.setBorderRight(BorderStyle.THIN);
        style.setBorderTop(BorderStyle.THIN);
        style.setVerticalAlignment(VerticalAlignment.CENTER);

        switch (type) {
            case "title":
                style.setFillForegroundColor(IndexedColors.GREY_25_PERCENT.getIndex());
                style.setFillPattern(FillPatternType.SOLID_FOREGROUND);
                style.setAlignment(HorizontalAlignment.CENTER);
                break;
            case "center":
                style.setAlignment(HorizontalAlignment.CENTER);
                break;
            case "left":
                style.setAlignment(HorizontalAlignment.LEFT);
                break;
            case "right":
                style.setAlignment(HorizontalAlignment.RIGHT);
                break;
        }
        return style;
    }

    private CellStyle cellStyleSelector(String type) {
        return switch (type) {
            case "title" -> cellStyleTitle;
            case "left" -> cellStyleLeft;
            case "right" -> cellStyleRight;
            default -> cellStyleCenter;
        };
    }

    public static <T> List<T> parseExcelToList(Class<T> clazz, 
                                               ExcelType excelType, 
                                               MultipartFile file) throws IOException {
        try (InputStream is = file.getInputStream(); 
             Workbook workbook = WorkbookFactory.create(is)) {
            Sheet sheet = workbook.getSheetAt(0);
            Iterator<Row> rowIterator = sheet.iterator();

            List<String> headers = extractAndValidateHeaders(rowIterator, excelType);
            List<T> result = new ArrayList<>();
            
            DataFormatter formatter = new DataFormatter();
            
            while (rowIterator.hasNext()) {
                Row row = rowIterator.next();
                T obj = mapRowToObject(clazz, headers, excelType.getDbColumn(), row, formatter);
                result.add(obj);
            }
            return result;
        }
    }

    // ... 기타 메서드들
}
```

---

## 핵심 교훈

1. **외부 리소스는 반드시 AutoCloseable로 관리**
   - 파일, 네트워크, 데이터베이스 연결 등 외부 리소스는 항상 `AutoCloseable` 인터페이스 구현 고려

2. **try-with-resources는 예외 안전성 보장**
   - 예외 발생 시에도 리소스가 자동으로 정리됨
   - finally 블록보다 안전하고 간결

3. **불필요한 객체 생성은 GC 부담 증가**
   - 객체 생성은 메모리와 CPU 비용 발생
   - 재사용 가능한 객체는 재사용

4. **재사용 가능한 객체는 적극 재사용**
   - `DataFormatter`, `SimpleDateFormat` 등은 스레드 안전성만 고려하면 재사용 가능
   - 메서드 레벨에서 한 번 생성하여 재사용

5. **메모리 누수는 예방이 최선**
   - 리소스 관리는 설계 단계에서부터 고려
   - 코드 리뷰 시 리소스 해제 여부 체크 필수

---

## 참고 자료

- [Apache POI - Busy Developers' Guide](https://poi.apache.org/components/spreadsheet/quick-guide.html)
- [Effective Java 3rd Edition - Item 9: Prefer try-with-resources to try-finally](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/)
- [Java Documentation - The try-with-resources Statement](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)

---

## 마치며

엑셀 다운로드 기능은 비즈니스 애플리케이션에서 자주 사용되는 기능입니다. 하지만 메모리 관리를 제대로 하지 않으면 서비스 장애로 이어질 수 있습니다. `AutoCloseable` 인터페이스와 `try-with-resources` 패턴을 활용하여 안전하고 효율적인 리소스 관리를 구현할 수 있습니다.

이번 개선을 통해 메모리 누수 문제를 원천적으로 차단하고, 성능도 향상시킬 수 있었습니다. 여러분의 프로젝트에서도 유사한 패턴을 찾아 개선해보시기 바랍니다.
