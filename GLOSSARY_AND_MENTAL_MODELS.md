# SimplyAdmission: Core Engineering Glossary & Mental Models

---

### 1. Idempotency
* **Definition**: An operation that can be applied multiple times without changing the result beyond the initial application.
* **CRM Context**: If a school's payment gateway retries a "Payment Succeeded" webhook 5 times due to a network glitch, SimplyAdmission processes the fee once and ignores the next 4 requests.

### 2. Transactional Outbox Pattern
* **Definition**: A pattern that guarantees atomic state updates between a database write and a message broker publication without distributed 2-phase commits (XA).
* **CRM Context**: When an applicant's stage changes to `ENROLLED`, an event is written to an `outbox_events` table in the *same DB transaction*. A background worker reads that table and pushes the event to Kafka, ensuring zero event loss even if Kafka is temporarily down.

### 3. Virtual Threads (Project Loom)
* **Definition**: Extremely lightweight user-mode threads managed directly by the Java Virtual Machine rather than the operating system.
* **CRM Context**: Enables SimplyAdmission to handle thousands of concurrent outbound HTTP calls (SMS/WhatsApp notifications, telephony webhooks) with simple synchronous blocking code (`Thread.sleep`, `HttpClient.send`), consuming a fraction of the RAM required by traditional platform threads.

### 4. MVCC (Multi-Version Concurrency Control)
* **Definition**: A database mechanism where readers do not block writers, and writers do not block readers. PostgreSQL keeps multiple physical versions of a tuple/row.
* **CRM Context**: Counselors querying dashboard metrics do not block parents who are submitting new admission applications.

### 5. Circuit Breaker (Resilience4j)
* **Definition**: A design pattern that prevents an application from repeatedly trying to execute an operation that's likely to fail.
* **CRM Context**: If an external school ERP system goes down, the circuit breaker trips to `OPEN`, immediately failing over or queuing sync requests instead of hanging SimplyAdmission threads.

### 6. Dead Letter Queue (DLQ)
* **Definition**: A secondary message queue holding messages that could not be successfully processed after reaching maximum retry thresholds.
* **CRM Context**: If a lead webhook contains malformed data or a third-party CRM sync fails repeatedly, the message is routed to the DLQ for manual inspection and alerting rather than blocking the main pipeline.
