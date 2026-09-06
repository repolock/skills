# Tutorial 29: Executive Summary Report Design & Development (World-Class Master Guide)

In an Admission CRM, school directors, trustees, and principals do not browse individual student profiles. They make multi-crore marketing, staffing, and infrastructure decisions based on **Executive Summary Reports**.

This guide covers the end-to-end design, database architecture, SQL aggregation, and Java development of enterprise summary reports.

---

## 1. The 4 Core Reporting Typologies in SimplyAdmission

```
┌────────────────────────────────────────────────────────────────────────┐
│                   SIMPLYADMISSION REPORTING MATRIX                     │
├────────────────────┬─────────────────────────────┬─────────────────────┤
│ Report Type        │ Audience                    │ Key Metrics         │
├────────────────────┼─────────────────────────────┼─────────────────────┤
│ 1. Operational     │ Admission Counselors & Team │ Daily Leads Called, │
│    Summary         │ Leads                       │ SLA Turnaround Time │
├────────────────────┼─────────────────────────────┼─────────────────────┤
│ 2. Funnel & Ad     │ Marketing Heads & Digital   │ Source-wise ROI,    │
│    Attribution     │ Agencies                    │ Drop-off Percentages│
├────────────────────┼─────────────────────────────┼─────────────────────┤
│ 3. Financial       │ Accounts Officers & Bursars │ Collections, Cash vs│
│    Summary         │                             │ Gateway, Pending Fee│
├────────────────────┼─────────────────────────────┼─────────────────────┤
│ 4. Executive Multi-│ Central School Society      │ Campus A vs B, Seat │
│    Campus Summary  │ Trustees & Directors        │ Occupancy, Revenue  │
└────────────────────┴─────────────────────────────┴─────────────────────┘
```

---

## 2. Database Architecture: The OLTP vs. OLAP Disaster

**The Danger**: A principal opens the dashboard on Monday morning and clicks *"Generate Annual 5-Year Admission Comparison"*.
* If this heavy query runs on your live transactional database (**OLTP**), PostgreSQL will lock tables, pin CPU at 100%, and parents currently submitting admission forms will experience timeouts and crashes!

```
                    ┌──────────────────────────────────────────────┐
                    │               WRITE OPERATIONS               │
                    │   (Parents submitting forms, paying fees)    │
                    └──────────────────────┬───────────────────────┘
                                           │
                                           ▼
                    ┌──────────────────────────────────────────────┐
                    │          PRIMARY DATABASE (OLTP)             │
                    │          PostgreSQL (Master Node)            │
                    └──────────────────────┬───────────────────────┘
                                           │ Streaming Replication (WAL)
                                           ▼
                    ┌──────────────────────────────────────────────┐
                    │         READ REPLICA DATABASE (OLAP)         │
                    │         PostgreSQL (Read-Only Node)          │
                    └──────────────────────┬───────────────────────┘
                                           │
                                           ▼
                    ┌──────────────────────────────────────────────┐
                    │          HEAVY REPORTING QUERIES             │
                    │   (Materialized Views, Grouping Sets)        │
                    └──────────────────────────────────────────────┘
```

### The Architectural Solution:
1. **Read Replicas (CQRS)**: Heavy reporting queries are routed to a read-only PostgreSQL replica.
2. **PostgreSQL Materialized Views**: Pre-calculating summary numbers in the background so dashboards load in **5 milliseconds** instead of 30 seconds!

---

## 3. Advanced SQL for Summary Reporting: `ROLLUP` & `GROUPING SETS`

Standard `GROUP BY` requires multiple queries to calculate sub-totals and grand totals. Advanced SQL solves this in **a single query**:

### A. The Multi-Campus Summary with `ROLLUP` (Subtotals & Grand Total)
```sql
SELECT 
    COALESCE(c.name, 'ALL CAMPUSES') AS campus_name,
    COALESCE(l.stage, 'TOTAL') AS stage,
    COUNT(l.id) AS total_students,
    SUM(p.amount) AS total_revenue
FROM campuses c
LEFT JOIN leads l ON l.campus_id = c.id
LEFT JOIN applications a ON a.lead_id = l.id
LEFT JOIN payments p ON p.application_id = a.id AND p.status = 'SUCCESS'
GROUP BY ROLLUP (c.name, l.stage);
```

### What `ROLLUP` Returns:
```
campus_name       stage          total_students    total_revenue
─────────────────────────────────────────────────────────────────
Vasant Kunj       NEW                      120             0.00
Vasant Kunj       ENROLLED                  45     11,25,000.00
Vasant Kunj       TOTAL                    165     11,25,000.00  <-- Campus Subtotal!
Gurgaon           NEW                      210             0.00
Gurgaon           ENROLLED                  80     20,00,000.00
Gurgaon           TOTAL                    290     20,00,000.00  <-- Campus Subtotal!
ALL CAMPUSES      TOTAL                    455     31,25,000.00  <-- GRAND TOTAL!
```

---

### B. High-Speed Pre-Aggregated Materialized Views
```sql
CREATE MATERIALIZED VIEW mv_daily_campus_admissions AS
SELECT 
    l.tenant_id,
    c.id AS campus_id,
    c.name AS campus_name,
    DATE_TRUNC('day', l.created_at) AS report_date,
    COUNT(l.id) AS total_inquiries,
    COUNT(CASE WHEN l.stage = 'ENROLLED' THEN 1 END) AS enrolled_count,
    ROUND(COUNT(CASE WHEN l.stage = 'ENROLLED' THEN 1 END)::numeric / NULLIF(COUNT(l.id), 0) * 100, 2) AS conversion_percentage,
    COALESCE(SUM(p.amount), 0) AS daily_revenue
FROM campuses c
JOIN leads l ON l.campus_id = c.id
LEFT JOIN applications a ON a.lead_id = l.id
LEFT JOIN payments p ON p.application_id = a.id AND p.status = 'SUCCESS'
GROUP BY l.tenant_id, c.id, c.name, DATE_TRUNC('day', l.created_at);

-- Create unique index to allow non-blocking concurrent refresh!
CREATE UNIQUE INDEX idx_mv_daily_campus ON mv_daily_campus_admissions (campus_id, report_date);
```

### Refreshing without blocking reads:
```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_campus_admissions;
```

---

## 4. Java Backend Implementation: DTO Projections & Streaming Excel

### A. Spring Data JPA Projection (Zero Entity Overhead)
Instead of loading full Hibernate entities, project directly into a lightweight Java `record`:

```java
package com.simplyadmission.core.dto;

import java.math.BigDecimal;
import java.time.LocalDate;

public record CampusDailySummaryDTO(
    String campusName,
    LocalDate reportDate,
    long totalInquiries,
    long enrolledCount,
    double conversionPercentage,
    BigDecimal dailyRevenue
) {}
```

---

### B. Memory-Safe Streaming Excel Generation (Apache POI `SXSSF`)
Standard `XSSFWorkbook` loads the entire Excel sheet into heap memory, crashing the server on large exports.
**`SXSSFWorkbook` (Streaming POI)** flushes rows to temporary disk storage every 100 rows, consuming under 15MB of RAM!

```java
package com.simplyadmission.core.report;

import com.simplyadmission.core.dto.CampusDailySummaryDTO;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.streaming.SXSSFSheet;
import org.apache.poi.xssf.streaming.SXSSFWorkbook;

import java.io.OutputStream;
import java.util.List;

public class StreamingExcelReportGenerator {

    public static void generateSummaryExcel(List<CampusDailySummaryDTO> summaries, OutputStream outputStream) throws Exception {
        // Keep 100 rows in memory, flushing excess rows to disk!
        try (SXSSFWorkbook workbook = new SXSSFWorkbook(100)) {
            SXSSFSheet sheet = workbook.createSheet("Executive Summary");

            // 1. Header Row Styling
            CellStyle headerStyle = workbook.createCellStyle();
            Font font = workbook.createFont();
            font.setBold(true);
            headerStyle.setFont(font);
            headerStyle.setFillForegroundColor(IndexedColors.GREY_25_PERCENT.getIndex());
            headerStyle.setFillPattern(FillPatternType.SOLID_FOREGROUND);

            Row header = sheet.createRow(0);
            String[] columns = {"Campus", "Date", "Inquiries", "Enrolled", "Conversion %", "Revenue (INR)"};
            for (int i = 0; i < columns.length; i++) {
                Cell cell = header.createCell(i);
                cell.setCellValue(columns[i]);
                cell.setCellStyle(headerStyle);
            }

            // 2. Data Rows
            int rowIdx = 1;
            for (CampusDailySummaryDTO s : summaries) {
                Row row = sheet.createRow(rowIdx++);
                row.createCell(0).setCellValue(s.campusName());
                row.createCell(1).setCellValue(s.reportDate().toString());
                row.createCell(2).setCellValue(s.totalInquiries());
                row.createCell(3).setCellValue(s.enrolledCount());
                row.createCell(4).setCellValue(s.conversionPercentage() + "%");
                row.createCell(5).setCellValue(s.dailyRevenue().doubleValue());
            }

            workbook.write(outputStream);
            workbook.dispose(); // Cleans up temporary disk files!
        }
    }
}
```

---

## 5. Automated Scheduled Delivery (The 8:00 AM Morning Executive Digest)

Every morning at 8:00 AM, our backend automatically compiles the previous day's summary, generates the Excel and PDF attachments, and emails it to the Central Management Committee.

```java
package com.simplyadmission.core.scheduler;

import com.simplyadmission.core.report.StreamingExcelReportGenerator;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.io.ByteArrayOutputStream;

@Component
public class DailyExecutiveReportScheduler {

    // Runs every day at 08:00 AM (Cron expression: second minute hour day month weekday)
    @Scheduled(cron = "0 0 8 * * *")
    public void dispatchMorningExecutiveDigest() {
        System.out.println("⏰ [CRON 08:00 AM] Compiling Daily Executive Summary Digest...");

        // 1. Fetch pre-aggregated numbers from Materialized View
        var summaries = reportRepository.fetchDailySummaries();

        // 2. Generate Excel in memory
        ByteArrayOutputStream excelStream = new ByteArrayOutputStream();
        StreamingExcelReportGenerator.generateSummaryExcel(summaries, excelStream);

        // 3. Dispatch to School Directors via AWS SES
        emailService.sendReportEmail(
            "directors@dpssociety.edu",
            "📊 SimplyAdmission Executive Morning Digest - " + LocalDate.now(),
            excelStream.toByteArray(),
            "Daily_Admission_Digest.xlsx"
        );

        System.out.println("✅ Executive digest dispatched successfully!");
    }
}
```
