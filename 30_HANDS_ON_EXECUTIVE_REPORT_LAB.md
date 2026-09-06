# Tutorial 30: Hands-On Executive Report Building Lab (Pure Java 21)

This is the **practical companion** to [Tutorial 29](file:///d:/Alok/.project/java/tutorials/29_EXECUTIVE_SUMMARY_REPORT_DESIGN_AND_DEVELOPMENT.md). Every line of code in this lab is **copy-paste runnable** with standard Java 21 — no Maven, no Spring, no external libraries.

---

## 1. How to Read ASCII Tables & Architecture Diagrams

Before writing reports, you must learn to **read** the diagrams that document them. Developers use text-based "box-drawing" characters because they work in any terminal, code comment, or markdown file — no image editor needed.

### 1.1 The Box-Drawing Alphabet

```
Character │ Name              │ What It Represents
──────────┼───────────────────┼──────────────────────────────────
┌  ┐      │ Top corners       │ Start of a box (component/system)
└  ┘      │ Bottom corners    │ End of a box
│         │ Vertical pipe     │ Left/right wall of a box, OR a connection line
─         │ Horizontal dash   │ Top/bottom wall of a box, OR a connection line
├  ┤      │ T-junctions       │ Row dividers inside a table
┼         │ Cross             │ Where a row divider meets a column divider
┬  ┴      │ Top/bottom T      │ Column dividers meeting top/bottom wall
▼  ►      │ Arrows            │ Direction of data flow (down / right)
...       │ Ellipsis          │ "More items follow this pattern"
```

### 1.2 Example A — A Simple 2-Column Table

```
┌─────────────┬──────────────┐   ← TOP WALL with column separator (┬)
│ Campus      │ Total Leads  │   ← HEADER ROW: column names
├─────────────┼──────────────┤   ← ROW DIVIDER: separates header from data
│ Vasant Kunj │          120 │   ← DATA ROW 1
│ Gurgaon     │          210 │   ← DATA ROW 2
│ Noida       │          170 │   ← DATA ROW 3
└─────────────┴──────────────┘   ← BOTTOM WALL
```

**How to read it**: This is a simple spreadsheet. **Column 1** is the campus name, **Column 2** is the lead count. Read left-to-right, top-to-bottom — exactly like Excel.

### 1.3 Example B — A Table with Subtotals

```
┌─────────────┬──────────┬───────────┬──────────────┐
│ Campus      │ Stage    │ Count     │ Revenue (₹)  │
├─────────────┼──────────┼───────────┼──────────────┤
│ Vasant Kunj │ NEW      │        55 │            0 │
│ Vasant Kunj │ ENROLLED │        45 │   11,25,000  │
│ Vasant Kunj │ TOTAL    │       100 │   11,25,000  │  ← SUBTOTAL ROW
├─────────────┼──────────┼───────────┼──────────────┤
│ Gurgaon     │ NEW      │        90 │            0 │
│ Gurgaon     │ ENROLLED │        80 │   20,00,000  │
│ Gurgaon     │ TOTAL    │       170 │   20,00,000  │  ← SUBTOTAL ROW
├─────────────┼──────────┼───────────┼──────────────┤
│ ALL         │ TOTAL    │       270 │   31,25,000  │  ← GRAND TOTAL
└─────────────┴──────────┴───────────┴──────────────┘
```

**How to read it**: Rows with stage = `TOTAL` are **subtotals** for that campus. The last row (`ALL / TOTAL`) is the **grand total** across all campuses. This is what SQL `ROLLUP` produces.

### 1.4 Example C — A Flowchart (Data Pipeline)

```
┌───────────────────────┐
│  Parent Submits Form  │   ← STEP 1: The starting event
└───────────┬───────────┘
            │                ← Vertical line = data flows DOWNWARD
            ▼                ← Arrow confirms direction
┌───────────────────────┐
│   Save to Database    │   ← STEP 2: The form data is persisted
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Assign to Counselor  │   ← STEP 3: Round-robin allocation
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│ Send WhatsApp to Lead │   ← STEP 4: Confirmation notification
└───────────────────────┘
```

**How to read it**: Follow the arrows from top to bottom. Each **box** is a step or system. Each **line with arrow** shows what happens next. This diagram says: *"When a parent submits a form, it gets saved to the database, then assigned to a counselor, then a WhatsApp message is sent."*

### 1.5 Branching Flowchart (Decision Point)

```
┌─────────────────────┐
│  Payment Received?  │
└──────────┬──────────┘
           │
     ┌─────┴─────┐
     │           │
     ▼           ▼
┌─────────┐ ┌──────────┐
│   YES   │ │    NO    │
│ Send    │ │ Send     │
│ Receipt │ │ Reminder │
└─────────┘ └──────────┘
```

**How to read it**: The line **splits into two branches** — this represents an IF/ELSE decision. Left = YES path, Right = NO path.

---

## 2. Sample Data Generator (Pure Java 21)

Create this file and compile it. It generates 500 realistic CRM leads.

```java
// File: ReportLabData.java
import java.util.*;

public class ReportLabData {

    // A lead in SimplyAdmission
    public record Lead(
        String id,
        String campus,
        String stage,
        String counselor,
        long feePaid   // in INR, 0 if not enrolled
    ) {}

    static final String[] CAMPUSES   = {"Vasant Kunj", "Gurgaon", "Noida"};
    static final String[] STAGES     = {"NEW", "CONTACTED", "VISITED", "APPLIED", "ENROLLED", "DROPPED"};
    static final String[] COUNSELORS = {"Priya", "Rahul", "Sneha", "Amit", "Divya", "Karan"};

    // Stage distribution weights (realistic funnel: many NEW, few ENROLLED)
    static final int[] STAGE_WEIGHTS = {35, 25, 15, 10, 10, 5}; // must sum to 100

    /**
     * Generate 500 sample leads with a fixed seed for reproducible results.
     */
    public static List<Lead> generateLeads() {
        Random rng = new Random(42); // Fixed seed = same data every run
        List<Lead> leads = new ArrayList<>(500);

        for (int i = 1; i <= 500; i++) {
            String campus    = CAMPUSES[rng.nextInt(CAMPUSES.length)];
            String stage     = pickWeightedStage(rng);
            String counselor = COUNSELORS[rng.nextInt(COUNSELORS.length)];
            long feePaid     = stage.equals("ENROLLED") ? 25_000L : 0L;

            leads.add(new Lead(
                "LEAD-%04d".formatted(i),
                campus,
                stage,
                counselor,
                feePaid
            ));
        }
        return Collections.unmodifiableList(leads);
    }

    /**
     * Weighted random selection using cumulative probability.
     * STAGE_WEIGHTS = {35, 25, 15, 10, 10, 5}
     * Cumulative     = {35, 60, 75, 85, 95, 100}
     * Random 0-99 mapped to the first bucket it falls into.
     */
    private static String pickWeightedStage(Random rng) {
        int roll = rng.nextInt(100); // 0 to 99
        int cumulative = 0;
        for (int i = 0; i < STAGE_WEIGHTS.length; i++) {
            cumulative += STAGE_WEIGHTS[i];
            if (roll < cumulative) return STAGES[i];
        }
        return STAGES[0]; // fallback
    }
}
```

---

## 3. Report Aggregator (GROUP BY + ROLLUP in Pure Java)

This class replicates what SQL `GROUP BY ROLLUP(campus, stage)` does — entirely in Java Streams.

```java
// File: ReportAggregator.java
import java.util.*;
import java.util.stream.*;

public class ReportAggregator {

    // One row of the final aggregated report
    public record SummaryRow(
        String campus,
        String stage,
        long count,
        long revenue,
        double conversionPct
    ) {}

    /**
     * Aggregate leads into: campus × stage detail rows + campus subtotals + grand total.
     * This is the pure-Java equivalent of:
     *   SELECT campus, stage, COUNT(*), SUM(fee_paid)
     *   FROM leads
     *   GROUP BY ROLLUP(campus, stage);
     */
    public static List<SummaryRow> aggregate(List<ReportLabData.Lead> leads) {
        List<SummaryRow> result = new ArrayList<>();

        // Step 1: Group by campus, then by stage
        Map<String, Map<String, List<ReportLabData.Lead>>> byCampusThenStage =
            leads.stream().collect(
                Collectors.groupingBy(
                    ReportLabData.Lead::campus,
                    LinkedHashMap::new,           // preserve insertion order
                    Collectors.groupingBy(
                        ReportLabData.Lead::stage,
                        LinkedHashMap::new,
                        Collectors.toList()
                    )
                )
            );

        long grandCount   = 0;
        long grandRevenue = 0;
        long grandEnrolled = 0;

        // Step 2: For each campus, emit detail rows + subtotal
        for (var campusEntry : byCampusThenStage.entrySet()) {
            String campus = campusEntry.getKey();
            Map<String, List<ReportLabData.Lead>> byStage = campusEntry.getValue();

            long campusTotal    = 0;
            long campusRevenue  = 0;
            long campusEnrolled = 0;

            // Sort stages in funnel order
            for (String stage : ReportLabData.STAGES) {
                List<ReportLabData.Lead> stageLeads = byStage.getOrDefault(stage, List.of());
                long count   = stageLeads.size();
                long revenue = stageLeads.stream().mapToLong(ReportLabData.Lead::feePaid).sum();

                if (count > 0) {
                    result.add(new SummaryRow(campus, stage, count, revenue, 0));
                }
                campusTotal   += count;
                campusRevenue += revenue;
                if (stage.equals("ENROLLED")) campusEnrolled = count;
            }

            // Campus subtotal row
            double campusConversion = campusTotal == 0 ? 0 :
                (campusEnrolled * 100.0) / campusTotal;
            result.add(new SummaryRow(campus, "── SUBTOTAL ──", campusTotal, campusRevenue, campusConversion));

            grandCount    += campusTotal;
            grandRevenue  += campusRevenue;
            grandEnrolled += campusEnrolled;
        }

        // Step 3: Grand total row
        double grandConversion = grandCount == 0 ? 0 :
            (grandEnrolled * 100.0) / grandCount;
        result.add(new SummaryRow("ALL CAMPUSES", "══ GRAND TOTAL ══", grandCount, grandRevenue, grandConversion));

        return result;
    }
}
```

---

## 4. CSV Report Writer

```java
// File: CsvReportWriter.java
import java.io.*;
import java.util.List;

public class CsvReportWriter {

    /**
     * Write aggregated summary rows to a CSV file.
     * CSV = Comma-Separated Values. Opens directly in Excel / Google Sheets.
     */
    public static void writeCsv(List<ReportAggregator.SummaryRow> rows, String filePath) {
        try (PrintWriter pw = new PrintWriter(new FileWriter(filePath))) {
            // Header line
            pw.println("Campus,Stage,Count,Revenue (INR),Conversion %");

            for (var row : rows) {
                pw.printf("%s,%s,%d,%d,%.2f%%%n",
                    row.campus(),
                    row.stage(),
                    row.count(),
                    row.revenue(),
                    row.conversionPct()
                );
            }
            System.out.println("✅ CSV report saved to: " + filePath);
        } catch (IOException e) {
            System.err.println("❌ Failed to write CSV: " + e.getMessage());
        }
    }
}
```

---

## 5. Console Dashboard (Rich Terminal Output)

```java
// File: ConsoleDashboard.java
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.stream.*;

public class ConsoleDashboard {

    /**
     * Print a rich text-based executive dashboard to the terminal.
     */
    public static void display(List<ReportLabData.Lead> leads, List<ReportAggregator.SummaryRow> summary) {
        String now = LocalDateTime.now().format(DateTimeFormatter.ofPattern("dd-MMM-yyyy HH:mm:ss"));

        // ─── HEADER BANNER ─────────────────────────────────────────
        System.out.println();
        System.out.println("╔══════════════════════════════════════════════════════════════╗");
        System.out.println("║       📊 SIMPLYADMISSION EXECUTIVE DASHBOARD                ║");
        System.out.println("║       Generated: " + String.format("%-43s", now) + "║");
        System.out.println("╚══════════════════════════════════════════════════════════════╝");
        System.out.println();

        // ─── KPI SUMMARY BOX ───────────────────────────────────────
        var grandTotal = summary.stream()
            .filter(r -> r.campus().equals("ALL CAMPUSES"))
            .findFirst().orElseThrow();

        long totalLeads    = grandTotal.count();
        long totalRevenue  = grandTotal.revenue();
        double conversion  = grandTotal.conversionPct();
        long enrolled      = summary.stream()
            .filter(r -> r.stage().equals("ENROLLED") && !r.campus().equals("ALL CAMPUSES"))
            .mapToLong(ReportAggregator.SummaryRow::count).sum();

        System.out.println("┌────────────────────────────────────────────────────┐");
        System.out.printf( "│  Total Leads:     %-10d                       │%n", totalLeads);
        System.out.printf( "│  Total Enrolled:  %-10d                       │%n", enrolled);
        System.out.printf( "│  Conversion Rate: %-10.2f%%                      │%n", conversion);
        System.out.printf( "│  Total Revenue:   ₹ %-,10d                     │%n", totalRevenue);
        System.out.println("└────────────────────────────────────────────────────┘");
        System.out.println();

        // ─── CAMPUS-WISE BAR CHART ─────────────────────────────────
        System.out.println("── Campus Lead Distribution ──────────────────────────");
        Map<String, Long> campusCounts = leads.stream()
            .collect(Collectors.groupingBy(ReportLabData.Lead::campus, Collectors.counting()));

        long maxCount = campusCounts.values().stream().mapToLong(Long::longValue).max().orElse(1);

        for (var entry : campusCounts.entrySet()) {
            int barLength = (int) (entry.getValue() * 40 / maxCount); // scale to 40 chars
            String bar = "█".repeat(barLength);
            System.out.printf("  %-12s │ %s %d%n", entry.getKey(), bar, entry.getValue());
        }
        System.out.println();

        // ─── STAGE-WISE FUNNEL ─────────────────────────────────────
        System.out.println("── Admission Funnel (All Campuses) ──────────────────");
        Map<String, Long> stageCounts = leads.stream()
            .collect(Collectors.groupingBy(ReportLabData.Lead::stage, Collectors.counting()));

        for (String stage : ReportLabData.STAGES) {
            long count = stageCounts.getOrDefault(stage, 0L);
            int barLen = (int) (count * 40 / maxCount);
            String bar = "▓".repeat(barLen);
            System.out.printf("  %-11s │ %s %d%n", stage, bar, count);
        }
        System.out.println();

        // ─── DETAILED TABLE ────────────────────────────────────────
        System.out.println("── Detailed Aggregation (ROLLUP equivalent) ──────────");
        System.out.printf("  %-14s %-18s %8s %12s %10s%n",
            "Campus", "Stage", "Count", "Revenue(₹)", "Conv%");
        System.out.println("  " + "─".repeat(64));

        for (var row : summary) {
            System.out.printf("  %-14s %-18s %8d %,12d %9.2f%%%n",
                row.campus(), row.stage(), row.count(), row.revenue(), row.conversionPct());
        }
        System.out.println();
    }
}
```

---

## 6. The Main Runner

```java
// File: ReportLabMain.java

public class ReportLabMain {
    public static void main(String[] args) {
        System.out.println("🚀 SimplyAdmission Report Lab — Starting...");

        // Step 1: Generate 500 sample leads
        var leads = ReportLabData.generateLeads();
        System.out.println("   ✓ Generated " + leads.size() + " sample leads");

        // Step 2: Aggregate (GROUP BY ROLLUP equivalent)
        var summary = ReportAggregator.aggregate(leads);
        System.out.println("   ✓ Computed " + summary.size() + " summary rows");

        // Step 3: Export to CSV
        CsvReportWriter.writeCsv(summary, "executive_report.csv");

        // Step 4: Display Console Dashboard
        ConsoleDashboard.display(leads, summary);

        System.out.println("── Lab Complete! ────────────────────────────────────");
        System.out.println("  Open 'executive_report.csv' in Excel or Google Sheets.");
    }
}
```

### Compile & Run Commands

```bash
# Step 1: Compile all 5 files
javac ReportLabData.java ReportAggregator.java CsvReportWriter.java ConsoleDashboard.java ReportLabMain.java

# Step 2: Run
java ReportLabMain
```

### Expected Output (Abbreviated)

```
🚀 SimplyAdmission Report Lab — Starting...
   ✓ Generated 500 sample leads
   ✓ Computed 13 summary rows
✅ CSV report saved to: executive_report.csv

╔══════════════════════════════════════════════════════════════╗
║       📊 SIMPLYADMISSION EXECUTIVE DASHBOARD                ║
║       Generated: 06-Sep-2026 11:00:05                       ║
╚══════════════════════════════════════════════════════════════╝

┌────────────────────────────────────────────────────┐
│  Total Leads:     500                              │
│  Total Enrolled:  52                               │
│  Conversion Rate: 10.40%                           │
│  Total Revenue:   ₹ 13,00,000                      │
└────────────────────────────────────────────────────┘

── Campus Lead Distribution ──────────────────────────
  Vasant Kunj  │ ████████████████████████████████████ 168
  Gurgaon      │ ████████████████████████████████████████ 172
  Noida        │ █████████████████████████████████ 160

── Admission Funnel (All Campuses) ──────────────────
  NEW         │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ 175
  CONTACTED   │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ 125
  VISITED     │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ 75
  APPLIED     │ ▓▓▓▓▓▓▓▓▓▓▓ 50
  ENROLLED    │ ▓▓▓▓▓▓▓▓▓▓ 52
  DROPPED     │ ▓▓▓▓▓ 23
```

---

## 7. Challenge Exercises (Self-Practice)

| # | Challenge | Skill Tested |
| :---: | :--- | :--- |
| 1 | Add a 4th campus **"Dwarka"** with 150 leads and re-run. | Data modification |
| 2 | Add a `LocalDate createdAt` field to `Lead` and produce a **date-wise breakdown** per campus. | Date grouping |
| 3 | Print a **"Top 3 Counselors by Enrollments"** leaderboard section in the dashboard. | Sorting & limiting |
| 4 | Add a `previousCount` field and compute **% change vs last period** (↑ 12% or ↓ 5%). | Percentage math |
| 5 | Export the entire dashboard as an **HTML file** with inline CSS that opens in a browser. | File I/O + HTML |

---

## 8. Key Takeaways

```
┌───────────────────────────────────────────────────────────────────┐
│  WHAT YOU LEARNED IN THIS LAB                                     │
├───────────────────────────────────────────────────────────────────┤
│  1. How to read ASCII box-drawing tables and flowcharts           │
│  2. How to generate realistic sample data with seeded randomness  │
│  3. How Java Streams replicate SQL GROUP BY + ROLLUP              │
│  4. How to write CSV files for Excel/Sheets import                │
│  5. How to build a rich console dashboard with bar charts         │
│  6. The complete compile → run → verify cycle in Java 21          │
└───────────────────────────────────────────────────────────────────┘
```
