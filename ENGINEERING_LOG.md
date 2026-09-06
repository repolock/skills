# SimplyAdmission: Engineering & Architectural Decision Log

This is our living engineering journal. Every architectural decision, performance optimization, and production bug we fix is recorded here to build long-term retention.

---

## 📌 Log Entry Template
```markdown
### [YYYY-MM-DD] - Sprint X, Day Y: <Topic Name>
* **Problem / Goal**: What were we trying to solve or build?
* **Architectural Trade-off**: What options were considered and why did we pick this approach?
* **Bug / Challenge Encountered**: What failed, broke, or threw an error?
* **Root Cause**: Why did it fail? (JVM memory, race condition, lock contention, query plan, etc.)
* **Solution Applied**: How did we resolve it cleanly?
* **Key Code Pattern / Takeaway**: The core design pattern or principle learned.
```

---

## 🚀 Log Entries

### [2026-09-06] - Sprint 1, Day 1: Project Initialization & Domain Modeling
* **Goal**: Establish repository foundations, domain architecture, and foundational mental models for SimplyAdmission CRM.
* **Architectural Trade-off**: Chose an Inverted Learning Model (Project-First, TDD) over passive documentation reading to build real muscle memory.
* **Initial Files Created**:
  - `SYSTEM_DESIGN.md`: Full entity dictionary and ingestion pipeline.
  - `PRODUCTION_FAILURE_CATALOG.md`: Database and distributed system traps.
  - `GLOSSARY_AND_MENTAL_MODELS.md`: Key enterprise concepts.
  - `ENGINEERING_LOG.md`: This living record.
* **Next Target**: Verify local Java 21 toolchain and implement `Lead` / `Counselor` domain models with JUnit 5 concurrent allocation tests.
