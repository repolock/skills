# Tutorial 15: Omnichannel Communication Engine (WhatsApp, SMS & HTML Email)

In SimplyAdmission, parents expect immediate communication:
1. **WhatsApp**: Welcome brochure, interview slot reminders, and offer letters.
2. **SMS**: Transactional OTPs for login and fee payment alerts.
3. **Email**: Rich, branded HTML admission notices and receipt attachments.

This tutorial covers how to build a unified notification engine in Java with **Priority Queue Decoupling** and **Virtual Threads**.

---

## 1. The Priority Queue Architecture (Why Direct Calling Fails)

**The Disaster**: A school sends a marketing blast to 20,000 inquiries at 10:00 AM. At 10:01 AM, a parent requests an OTP to log in.
* **Without Queues**: The OTP is stuck at the end of the 20,000-message line! The parent waits 15 minutes, gets frustrated, and abandons the form.
* **With Priority Queues**: Urgent OTPs enter **Priority 0 (P0)**, while marketing blasts enter **Priority 2 (P2)**. P0 messages bypass the line instantly!

```
                    ┌────────────────────────────────────────────────────────┐
                    │                    MESSAGE PRODUCER                    │
                    └───────────────────────────┬────────────────────────────┘
                                                │
                 ┌──────────────────────────────┴──────────────────────────────┐
                 ▼ (High Priority: OTPs)                                       ▼ (Low Priority: Bulk Newsletters)
       ┌──────────────────┐                                          ┌──────────────────┐
       │ Queue: P0-URGENT │                                          │ Queue: P2-BULK   │
       └─────────┬────────┘                                          └─────────┬────────┘
                 │                                                             │
                 ▼                                                             ▼
       ┌────────────────────────────────────────────────────────────────────────────────┐
       │                WORKER POOL (Java 21 Virtual Threads - Loom)                    │
       └─────────────────┬───────────────────────┬──────────────────────────────┬───────┘
                         │                       │                              │
                         ▼                       ▼                              ▼
                 ┌───────────────┐       ┌───────────────┐              ┌───────────────┐
                 │ Meta WhatsApp │       │ Twilio / SMS  │              │    AWS SES    │
                 │   Cloud API   │       │    Gateway    │              │  (HTML Email) │
                 └───────────────┘       └───────────────┘              └───────────────┘
```

---

## 2. Channel 1: WhatsApp Business Cloud API (Meta)

To prevent spam, Meta does **not** allow businesses to send arbitrary text to users who haven't messaged first. You must send pre-approved **Message Templates** with dynamic parameters.

### WhatsApp Cloud API Request in Java:
```java
package com.simplyadmission.core.notification;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

public class WhatsAppNotificationService {

    private static final String META_WA_URL = "https://graph.facebook.com/v19.0/YOUR_PHONE_NUMBER_ID/messages";
    private final String bearerToken;

    public WhatsAppNotificationService(String bearerToken) {
        this.bearerToken = bearerToken;
    }

    public boolean sendAdmissionWelcomeTemplate(String recipientPhone, String parentName, String studentName) {
        // WhatsApp template: "Hello {{1}}, thank you for inquiring for {{2}} at DPS."
        String jsonPayload = """
        {
          "messaging_product": "whatsapp",
          "to": "%s",
          "type": "template",
          "template": {
            "name": "admission_welcome_v1",
            "language": { "code": "en" },
            "components": [
              {
                "type": "body",
                "parameters": [
                  { "type": "text", "text": "%s" },
                  { "type": "text", "text": "%s" }
                ]
              }
            ]
          }
        }
        """.formatted(recipientPhone, parentName, studentName);

        try {
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(META_WA_URL))
                .header("Content-Type", "application/json")
                .header("Authorization", "Bearer " + bearerToken)
                .POST(HttpRequest.BodyPublishers.ofString(jsonPayload))
                .build();

            HttpResponse<String> res = HttpClient.newHttpClient().send(request, HttpResponse.BodyHandlers.ofString());
            return res.statusCode() == 200;
        } catch (Exception e) {
            return false;
        }
    }
}
```

---

## 3. Channel 2: Dynamic HTML Email Rendering (Thymeleaf + AWS SES)

Sending ugly plain-text emails looks unprofessional for premier schools. We use **Thymeleaf Template Engine** to compile dynamic HTML:

### The Template: `templates/email/offer-letter.html`
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <style>
        .card { font-family: 'Segoe UI', sans-serif; max-width: 600px; margin: 0 auto; border: 1px solid #e2e8f0; border-radius: 8px; padding: 24px; }
        .btn { background: #2563eb; color: white; padding: 12px 24px; text-decoration: none; border-radius: 6px; display: inline-block; font-weight: bold; }
    </style>
</head>
<body>
    <div class="card">
        <h2 style="color: #1e3a8a;">🎉 Congratulations, <span th:text="${parentName}">Parent</span>!</h2>
        <p>We are delighted to offer admission for <strong th:text="${studentName}">Aarav</strong> into 
           <strong th:text="${grade}">Grade 5</strong> at Delhi Public School for academic year 2026-2027.</p>
        <p>Please confirm the seat by paying the registration deposit before the cut-off date: 
           <strong style="color: #dc2626;" th:text="${deadline}">March 15, 2026</strong>.</p>
        <div style="text-align: center; margin: 30px 0;">
            <a th:href="${paymentLink}" class="btn">Pay Admission Deposit Online</a>
        </div>
    </div>
</body>
</html>
```

### Compiling and Sending in Java:
```java
package com.simplyadmission.core.notification;

import org.thymeleaf.TemplateEngine;
import org.thymeleaf.context.Context;

public class EmailCompilerService {

    private final TemplateEngine templateEngine;

    public EmailCompilerService(TemplateEngine templateEngine) {
        this.templateEngine = templateEngine;
    }

    public String renderOfferEmail(String parentName, String studentName, String grade, String deadline, String link) {
        Context context = new Context();
        context.setVariable("parentName", parentName);
        context.setVariable("studentName", studentName);
        context.setVariable("grade", grade);
        context.setVariable("deadline", deadline);
        context.setVariable("paymentLink", link);

        // Compiles template + variables into complete HTML string
        return templateEngine.process("email/offer-letter", context);
    }
}
```

---

## 4. Superpower: Virtual Threads for Concurrent Notification Blasts

In Java 21, dispatching 5,000 notifications is trivial with **Virtual Threads**:

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Lead lead : pendingLeads) {
        executor.submit(() -> {
            whatsAppService.sendAdmissionWelcomeTemplate(lead.phoneNumber(), lead.parentName(), lead.studentName());
        });
    }
} // Automatically waits for all 5,000 virtual threads to finish!
```
Virtual threads do not block OS resources while waiting for Meta or Twilio to answer, utilizing less than 50MB of RAM for the entire batch!
