# Tutorial 19: Change Management, Impact Analysis & Release Notes

In an enterprise CRM, you never ship code blindly. A change to a database column or an API response can break the mobile app, drop leads from Meta ads, or corrupt accountant financial reports.

This guide provides the industry frameworks for:
1. **Feature Impact Area Analysis**.
2. **Writing Professional Release Notes (Engineering vs. Customer vs. Ops)**.
3. **Creating Standard Operating Procedures (SOPs) for Operations & Support Teams**.

---

## 1. The 5-Dimension Feature Impact Matrix

Before writing code for a feature (e.g., *"Add Sibling Discount to Application Fee"*), every developer must fill out this **Impact Analysis**:

```
┌────────────────────────────────────────────────────────────────────────┐
│                   FEATURE IMPACT ANALYSIS TEMPLATE                     │
├────────────────────┬───────────────────────────────────────────────────┤
│ Dimension          │ Impact Assessment & Questions                     │
├────────────────────┼───────────────────────────────────────────────────┤
│ 1. Database        │ • Does this require a Flyway migration?           │
│                    │ • Will it lock a heavy table during execution?    │
│                    │ • Are indexes needed for new search filters?      │
├────────────────────┼───────────────────────────────────────────────────┤
│ 2. API Contracts   │ • Is this breaking backward compatibility?        │
│                    │ • Did we rename or delete any JSON keys?          │
│                    │ • Will mobile apps running older versions crash?   │
├────────────────────┼───────────────────────────────────────────────────┤
│ 3. External Sync   │ • Does this affect webhooks (Meta/Razorpay)?      │
│                    │ • Does the School ERP schema support this field?  │
├────────────────────┼───────────────────────────────────────────────────┤
│ 4. Workflows       │ • Does this change counselor round-robin rules?   │
│                    │ • Will parents receive different automated SMS?   │
├────────────────────┼───────────────────────────────────────────────────┤
│ 5. Rollback Plan   │ • If this feature fails in production at 10 AM,   │
│                    │   how do we turn it off in 60 seconds? (Feature   │
│                    │   flag / DB rollback script).                     │
└────────────────────┴───────────────────────────────────────────────────┘
```

---

## 2. Professional Release Notes Framework

Release notes bridge the gap between developers, school principals, and internal ops teams. A single release produces **two different versions**:

### Version A: Internal Technical Release Notes (For Engineers & DevOps)
```markdown
# Release v2.4.0 (Build 894) - 2026-03-15

### 🚀 Database Migrations
- `V12__add_sibling_discount_to_applications.sql`: Adds `sibling_discount_applied` (BOOLEAN DEFAULT FALSE) and GIN index update on `custom_data`.

### ⚡ API Changes
- `POST /api/v1/applications/{id}/calculate-fee`: Added optional field `siblingRollNumber`.
- [NON-BREAKING]: Response payload now includes `discountBreakdown` object.

### 🐛 Bug Fixes
- Fixed race condition where two counselors claimed the same lead simultaneously by introducing `SELECT ... FOR UPDATE SKIP LOCKED` in `LeadAllocationService.java`.

### 🔄 Rollback Procedure
Run `flyway undo` to revert migration V12. Toggle feature flag `crm.features.sibling-discount=false` in Spring Cloud Config.
```

### Version B: Operations & Customer-Facing Release Notes (For Schools & Counselors)
```markdown
# What's New in SimplyAdmission - March 2026 Update! 🎓

### ✨ New Features
* **Automated Sibling Discount**: When entering an applicant's details, you can now enter their elder sibling's roll number. The system will automatically verify and apply a 15% discount on the admission fee!
* **Instant WhatsApp Fee Receipts**: Parents will now receive official GST receipts on WhatsApp immediately upon payment.

### 🛠️ Improvements
* **Faster Dashboard Loading**: Counselor lead lists now load in under 1 second, even during peak morning hours.
* **Refined Lead Status Filters**: You can now filter leads by both "Preferred Campus" and "Interview Date" at the same time.
```

---

## 3. Creating Training SOPs for Operations Teams

When engineering ships a new feature, the Operations team (Admissions Counselors, Callers, and Account Clerks) must know how to use it immediately.

### Standard Operating Procedure (SOP) Template:
```markdown
## SOP-ADM-042: How to Process Sibling Fee Waivers

### 🎯 Purpose:
Guide admission counselors on verifying and applying sibling discounts before issuing offer letters.

### 📋 Prerequisites:
* Role: `ADMISSION_COUNSELOR` or `CAMPUS_ADMIN`.
* Student status must be `UNDER_SCRUTINY`.

### 👣 Step-by-Step Instructions:
1. Navigate to **Admissions ➔ Applications** and open the student's profile.
2. Under the **Family Details** tab, check if the parent entered an elder sibling's name.
3. Click the new button: **🔍 Verify Sibling Record**.
   * If the elder sibling is actively enrolled, a green banner appears: *"Sibling Verified: Section 5-B"*.
   * The fee calculator will automatically adjust from ₹25,000 to ₹21,250.
4. Click **Issue Offer Letter**. An automated WhatsApp and Email alert will be dispatched to the parent.

### ⚠️ Troubleshooting / Edge Cases:
* **Sibling not found**: If the elder sibling changed sections recently, ask the accounts desk to verify the roll number in the School ERP before manually applying the discount.
* **Support Escalation**: If the verification button gives an error code `ERP_TIMEOUT_504`, raise a ticket to `#tech-support` on Slack with the student's Application ID.
```
