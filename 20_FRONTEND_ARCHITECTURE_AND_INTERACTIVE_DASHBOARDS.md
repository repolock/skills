# Tutorial 20: Frontend Architecture, Interactive Dashboards & Safe Custom Query Engine

A backend is only as valuable as the interface that connects it to humans. SimplyAdmission requires two completely different frontend experiences:
1. **The Public Applicant Portal**: Ultra-simple, mobile-optimized, fast-loading interface for parents.
2. **The Internal Management Dashboard**: Dense, data-heavy portal for school principals and counselors featuring charts, live filters, export tools, and custom query builders.

---

## 1. Dual-Portal Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 SIMPLYADMISSION FRONTEND                    │
├──────────────────────────────┬──────────────────────────────┤
│ 1. PUBLIC APPLICANT PORTAL   │ 2. INTERNAL CRM DASHBOARD    │
├──────────────────────────────┼──────────────────────────────┤
│ • Audience: Parents/Students │ • Audience: Counselors/Admin │
│ • Priority: Mobile speed     │ • Priority: High productivity│
│ • Features: Multi-step form, │ • Features: Kanban pipeline, │
│   document upload, Razorpay  │   call logs, charts, filters,│
│   modal, application status. │   bulk exports, custom query.│
└──────────────────────────────┴──────────────────────────────┘
```

---

## 2. Designing Interactive Dashboards (Filters, Graphs & Tables)

Management needs to answer questions in seconds:
> *"Show me how many Grade 5 leads from Instagram visited the campus this week versus last week."*

### The 4 Pillars of a High-Performance Dashboard:
1. **Dynamic Filter Bar**: Date range picker (`Today`, `This Week`, `Custom Range`), Campus dropdown, Stage multi-select, and Source filter (`Meta`, `Google`, `Walk-in`).
2. **KPI Metric Cards**: Total Inquiries, Conversion Rate (%), Total Revenue Collected (₹), Uncontacted Leads.
3. **Visual Graphs (Chart.js / Recharts)**:
   * **Funnel Drop-off Chart**: `Inquiry (1,000) ➔ Visited (450) ➔ Applied (200) ➔ Enrolled (95)`.
   * **Counselor Performance Bar Chart**: Number of calls made and leads converted per counselor.
4. **Server-Side Paginated Table**: Displays 25 records per page, sorting columns dynamically without fetching 500,000 rows to the browser!

### Frontend Dashboard HTML & Chart.js Implementation:
```html
<!-- Metric Cards Grid -->
<div class="kpi-grid">
    <div class="card"><h3>Total Inquiries</h3><p class="stat">1,420</p></div>
    <div class="card"><h3>Conversion Rate</h3><p class="stat">14.2%</p></div>
    <div class="card"><h3>Fees Collected</h3><p class="stat">₹ 42,50,000</p></div>
</div>

<!-- Interactive Chart Canvas -->
<div class="card">
    <h3>Admission Funnel Conversion</h3>
    <canvas id="funnelChart" width="400" height="150"></canvas>
</div>

<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script>
    const ctx = document.getElementById('funnelChart').getContext('2d');
    new Chart(ctx, {
        type: 'bar',
        data: {
            labels: ['Inquiries', 'Contacted', 'Campus Tour', 'Offer Issued', 'Enrolled'],
            datasets: [{
                label: 'Students',
                data: [1420, 980, 520, 310, 202],
                backgroundColor: ['#93c5fd', '#60a5fa', '#3b82f6', '#2563eb', '#1d4ed8']
            }]
        },
        options: { responsive: true, plugins: { legend: { display: false } } }
    });
</script>
```

---

## 3. Streaming Data Exports (CSV & Excel)

School administrators frequently click: **"Export to Excel"** to present numbers in board meetings.

### The Traps to Avoid:
* **Trap 1**: Loading 100,000 students into a Java `List<Lead>` will trigger `OutOfMemoryError`.
* **Trap 2**: Creating an Excel file in browser memory freezes the user's laptop.

### The Production Solution (Streaming HTTP Response):
The backend streams rows directly from PostgreSQL to the browser's download stream via `HttpServletResponse`:

```java
@GetMapping("/api/v1/leads/export/csv")
@PreAuthorize("hasAnyRole('CAMPUS_ADMIN', 'SUPER_ADMIN')")
public void exportLeadsToCsv(HttpServletResponse response) throws IOException {
    response.setContentType("text/csv");
    response.setHeader("Content-Disposition", "attachment; filename=leads_export.csv");

    PrintWriter writer = response.getWriter();
    writer.println("Student Name,Parent Phone,Grade,Stage,Counselor");

    // Spring Data JPA Stream reads rows one by one without loading all into RAM!
    try (var leadStream = leadRepository.streamAllByTenantId(TenantContextHolder.getContext().tenantId())) {
        leadStream.forEach(lead -> {
            writer.printf("%s,%s,%s,%s,%s%n",
                sanitize(lead.getStudentName()),
                lead.getParentPhone(),
                lead.getGradeApplied(),
                lead.getStage(),
                lead.getCounselorName()
            );
            writer.flush(); // Send chunks over network immediately!
        });
    }
}
```

---

## 4. The "Run Custom Query" Feature (Safe Report Builder)

Schools often ask for custom report builders:
> *"Can we write our own queries to filter leads?"*

### ⚠️ The Mortal Danger: Raw SQL Injection
If you give an end-user a text box to type SQL (`SELECT * FROM leads WHERE ...`), a disgruntled counselor or malicious user will enter:
```sql
'; DROP TABLE applications; --
```
Or they can enter queries that bypass multi-tenancy:
```sql
SELECT * FROM leads WHERE 1=1; -- Dumps competitor school's leads!
```

---

### 🛡️ The Enterprise Solution: Visual Filter Query Builder

Instead of raw SQL, give the user a **Structured Visual Rule Builder**:

```
[ Field: Stage       ▼ ]  [ Operator: EQUALS       ▼ ]  [ Value: CONTACTED   ]
[ AND / OR: AND       ]
[ Field: Lead Score  ▼ ]  [ Operator: GREATER_THAN ▼ ]  [ Value: 75          ]
[ AND / OR: AND       ]
[ Field: Bus Route   ▼ ]  [ Operator: IS_TRUE      ▼ ]
```

### How the Java Backend Executes It Safely:
The browser sends a structured JSON rule array. The backend compiles it using **Spring Data JPA Specifications**:

```json
{
  "rules": [
    { "field": "stage", "operator": "EQUALS", "value": "CONTACTED" },
    { "field": "engagementScore", "operator": "GREATER_THAN", "value": "75" }
  ],
  "limit": 500
}
```

### Safe Parameterized Criteria Compiler:
```java
package com.simplyadmission.core.query;

import com.simplyadmission.core.entity.LeadEntity;
import jakarta.persistence.criteria.Predicate;
import org.springframework.data.jpa.domain.Specification;
import java.util.ArrayList;
import java.util.List;

public class SafeQueryBuilder {

    public static Specification<LeadEntity> buildSpecification(List<QueryRule> rules, String tenantId) {
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();

            // 1. Mandatory Security Constraint (Zero bypass possible!)
            predicates.add(cb.equal(root.get("tenantId"), tenantId));

            // 2. Map only permitted safe whitelist fields
            for (QueryRule rule : rules) {
                switch (rule.field()) {
                    case "stage" -> predicates.add(cb.equal(root.get("stage"), rule.value()));
                    case "gradeApplied" -> predicates.add(cb.equal(root.get("gradeApplied"), rule.value()));
                    case "engagementScore" -> predicates.add(cb.greaterThan(root.get("engagementScore"), Integer.parseInt(rule.value())));
                    default -> throw new IllegalArgumentException("Unauthorized query field: " + rule.field());
                }
            }

            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }
}
```

### Why This is 100% Secure:
1. **Zero SQL Injection**: Hibernate automatically uses parameterized queries (`?`).
2. **Forced Multi-Tenancy**: The `tenant_id` predicate is hardcoded into the query builder.
3. **Strict Whitelist**: Users can only query pre-approved columns.
4. **Hard Limits**: The query execution enforces `PageRequest.of(0, Math.min(limit, 1000))` so no query can exhaust server RAM!
