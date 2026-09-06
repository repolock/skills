# SimplyAdmission: Production Failure & Anti-Pattern Catalog
### *How Real Backend Systems Break in Production and How We Prevent It*

---

## 1. 🛑 The N+1 Query Trap
* **Symptom**: Fetching 100 applications results in 101 SQL queries executed against PostgreSQL. Database CPU spikes to 100%, and API latency climbs from 20ms to 2.5s.
* **Root Cause**: Lazy loading on relationships (`@OneToMany documents`) without join fetching. Hibernate issues 1 query for the parent records, then iterates and executes 1 additional query per parent record to fetch its children.
* **Prevention**:
  1. Use `JOIN FETCH` in JPQL or `@EntityGraph(attributePaths = {"documents", "payments"})`.
  2. In CI/CD, use `datasource-proxy` or Hibernate query count assertions in integration tests to fail builds if query count > 1.

---

## 2. 🛑 The Webhook Replay Attack & Duplicate Payment Crediting
* **Symptom**: A parent is charged twice or an admission fee is marked as "Paid" multiple times due to gateway network retries or malicious replay attacks.
* **Root Cause**: Webhook endpoints treating incoming payloads without verifying cryptographic signatures or without idempotency controls.
* **Prevention**:
  1. **HMAC Signature Validation**: Compute SHA-256 HMAC of raw payload bytes against the gateway secret. Reject if signature doesn't match.
  2. **Idempotency Key via Redis**: Before processing, execute `SET payment:{gateway_tx_id} "PROCESSING" NX EX 60`. If the key exists, ignore or return cached `200 OK`.

---

## 3. 🛑 Database Connection Starvation (HikariCP Exhaustion)
* **Symptom**: API endpoints stop responding; thread dumps reveal dozens of threads waiting in `HikariPool.getConnection()` until timing out (`ConnectionTimeoutException`).
* **Root Cause**: Long-running non-database I/O (e.g., calling WhatsApp Business API, generating a heavy PDF, or calling a school ERP) inside an active `@Transactional` method. The database connection is held open while waiting for third-party network I/O.
* **Prevention**:
  1. **Keep Transactions Razor-Thin**: Only wrap pure database reads/writes in `@Transactional`.
  2. **Offload Third-Party Calls**: Dispatch events to Kafka/RabbitMQ or execute external calls via Java 21 Virtual Threads *after* the transaction commits.

---

## 4. 🛑 The Lead Grab Race Condition
* **Symptom**: Two counselors simultaneously click "Assign to Me" on a high-intent unassigned lead. Both counselors see a success message, and the student receives two conflicting phone calls.
* **Root Cause**: Non-atomic read-then-write sequence (`SELECT ...` followed by `UPDATE ...`) under Read Committed transaction isolation.
* **Prevention**:
  1. Use **Pessimistic Locking**: `SELECT * FROM leads WHERE id = :id FOR UPDATE SKIP LOCKED`.
  2. Or use **Optimistic Locking**: `@Version private Long version;` on the entity. The second update will fail with an `OptimisticLockException`.

---

## 5. 🛑 OutOfMemoryError (OOM) on Bulk Lead / Student Imports
* **Symptom**: A school admin uploads an Excel sheet with 50,000 historical students. The Spring Boot pod crashes with `java.lang.OutOfMemoryError: Java heap space`.
* **Root Cause**: Standard DOM parsing (like `new XSSFWorkbook(inputStream)` in Apache POI) loads the entire XML DOM of the spreadsheet into heap memory simultaneously (up to 50x the file size).
* **Prevention**:
  1. Use **Streaming Readers**: Use SAX event-driven streaming (`SXSSFWorkbook` or Alibaba `EasyExcel`).
  2. Stream rows one by one into batch inserts (e.g., 500 rows per batch) rather than loading all records into a `List`.
