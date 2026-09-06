# Tutorial 12: Marketing Attribution Mastery (UTM, GCLID, GTM, GA4 & Meta CAPI)

Schools spend lakhs of rupees on digital marketing campaigns across Google and Meta. Management needs to know:
> *"Did our ₹50,000 Google Search campaign actually result in paid admissions, or was our Facebook video ad better?"*

This requires **Full-Funnel Attribution Tracking** connecting frontend clicks to backend database records and sending offline conversion signals back to ad platforms.

---

## 1. The Attribution Landscape Cheat Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                        MARKETING ATTRIBUTION MAP                       │
├───────────────────┬─────────────────────────┬──────────────────────────┤
│ Technology        │ What It Does            │ Where It Lives           │
├───────────────────┼─────────────────────────┼──────────────────────────┤
│ UTM Parameters    │ Tags traffic source     │ Landing Page URL         │
│ GCLID             │ Google Click Identifier │ Landing Page URL ➔ DB    │
│ GTM               │ Tag Manager container   │ Frontend Browser         │
│ Meta Pixel ID     │ Client-side web tracker │ Frontend Browser         │
│ Meta CAPI         │ Server-side conversion  │ Java Backend ➔ Meta API  │
│ GA4 Protocol      │ Server-side analytics   │ Java Backend ➔ GA4 API   │
└───────────────────┴─────────────────────────┴──────────────────────────┘
```

---

## 2. UTM Parameters (Urchin Tracking Module)

When a parent clicks an ad, marketing tags are appended to the school's landing page URL:
```
https://dpsadmission.com/?utm_source=facebook&utm_medium=cpc&utm_campaign=admissions_2026&utm_content=video_tour
```

### The 5 Standard UTM Parameters:
1. **`utm_source`**: The platform sending traffic (e.g., `google`, `facebook`, `instagram`, `shiksha`).
2. **`utm_medium`**: The marketing channel (e.g., `cpc` [paid ads], `email`, `organic`).
3. **`utm_campaign`**: The specific promotion (e.g., `admissions_grade1_2026`).
4. **`utm_term`**: The target search keyword (e.g., `best_school_in_gurgaon`).
5. **`utm_content`**: Differentiates ad creative (e.g., `banner_blue` vs `video_tour`).

### Backend Action:
Your landing page JavaScript extracts these query parameters from `window.location.search` and places them into hidden input fields on the inquiry form. When submitted, the backend stores them in the `leads` table:
```sql
ALTER TABLE leads ADD COLUMN utm_source VARCHAR(100);
ALTER TABLE leads ADD COLUMN utm_medium VARCHAR(100);
ALTER TABLE leads ADD COLUMN utm_campaign VARCHAR(100);
```

---

## 3. GCLID (Google Click Identifier) & Offline Conversions

When someone clicks a Google Search Ad, Google appends a unique cryptographic click ID:
`?gclid=CjwKCAiA_...`

### The Problem:
A parent clicks a Google Ad on Monday, visits the campus on Wednesday, and pays the ₹50,000 admission fee on Friday **offline at the school accounts desk**. Google Ads has no idea the admission happened!

### The Solution (Google Offline Conversion Tracking):
1. When the parent fills the initial inquiry form, the frontend captures `gclid` and sends it to your Java backend.
2. The Java backend saves `gclid` in the database.
3. On Friday, when the accounts officer clicks "Payment Received", SimplyAdmission's backend calls the **Google Ads API**:
   > *"Hey Google! Click ID `CjwKCAiA_...` just converted for ₹50,000 in revenue!"*
4. Google Ads automatically optimizes its AI algorithm to find more high-paying parents!

---

## 4. GTM (Google Tag Manager) & GA4 Server-Side Measurement Protocol

* **Google Tag Manager (GTM)**: A code container on the school website that allows marketers to add tracking scripts without editing website code.
* **GA4 Measurement Protocol**: Allows your **Java backend** to send conversion events directly to Google Analytics 4 via HTTP POST, bypassing ad-blockers and Safari cookie restrictions!

### Sending an Enrollment Event from Java to GA4:
```java
public void trackEnrollmentInGA4(String clientId, double feeAmount) {
    String ga4Url = "https://www.google-analytics.com/mp/collect?api_secret=YOUR_SECRET&measurement_id=G-XXXXXX";

    String jsonPayload = """
    {
      "client_id": "%s",
      "events": [{
        "name": "admission_confirmed",
        "params": {
          "currency": "INR",
          "value": %.2f
        }
      }]
    }
    """.formatted(clientId, feeAmount);

    // Dispatch via Java HttpClient
}
```

---

## 5. Meta Pixel ID vs. Meta Conversions API (CAPI)

* **Meta Pixel (Client-Side)**: A JavaScript tag that runs in the browser.
  * *Major Problem*: Over 30% of browser pixel events are blocked by **Ad-blockers, iOS 14.5+ App Tracking Transparency (ATT), and cookie restrictions**.
* **Meta Conversions API / CAPI (Server-Side)**: The gold standard for modern CRMs. Your **Java server** talks directly to **Meta’s server** via an encrypted HTTP POST API. Ad-blockers cannot touch it!

### Privacy & Data Security: SHA-256 Hashing
Meta requires all personally identifiable information (PII) to be hashed using **SHA-256** before transmission:
* Email `rajesh@example.com` ➔ `b299e504e7...`
* Phone `+919876543210` ➔ `a48219ff01...`

### The Meta CAPI Java Dispatcher:
```java
package com.simplyadmission.core.attribution;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;

public class MetaCapiService {

    public static String hashSha256(String input) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(input.trim().toLowerCase().getBytes(StandardCharsets.UTF_8));
            StringBuilder hexString = new StringBuilder();
            for (byte b : hash) {
                String hex = Integer.toHexString(0xff & b);
                if (hex.length() == 1) hexString.append('0');
                hexString.append(hex);
            }
            return hexString.toString();
        } catch (Exception e) {
            throw new RuntimeException("SHA-256 hashing failed", e);
        }
    }

    public static String buildCapiPayload(String email, String phone, double feeAmount) {
        String hashedEmail = hashSha256(email);
        String hashedPhone = hashSha256(phone);

        return """
        {
          "data": [
            {
              "event_name": "Purchase",
              "event_time": %d,
              "action_source": "system_generated",
              "user_data": {
                "em": ["%s"],
                "ph": ["%s"]
              },
              "custom_data": {
                "currency": "INR",
                "value": %.2f
              }
            }
          ]
        }
        """.formatted(System.currentTimeMillis() / 1000L, hashedEmail, hashedPhone, feeAmount);
    }
}
```
