# SimplyAdmission: Core Engineering Repository

Welcome to the **SimplyAdmission Backend Core** development repository. This codebase serves as the real-world hands-on training ground for mastering enterprise Java backend engineering.

---

## 📁 Repository Structure
```
d:\Alok\.project\java
 ├── SYSTEM_DESIGN.md                  # Entity dictionary, pipelines & architecture rules
 ├── ENGINEERING_LOG.md                # Living log of architectural decisions & solved bugs
 ├── PRODUCTION_FAILURE_CATALOG.md     # Production anti-patterns (N+1, deadlocks, race conditions)
 ├── GLOSSARY_AND_MENTAL_MODELS.md     # Core engineering concepts (Outbox, Idempotency, MVCC)
 ├── pom.xml                           # Maven project descriptor (Java 21, JUnit 5, AssertJ, Mockito)
 └── src/
      ├── main/java/com/simplyadmission/core/
      │    ├── model/
      │    │    ├── Lead.java          # Immutable student inquiry record
      │    │    ├── LeadStage.java     # Finite state enum with transition checks
      │    │    └── Counselor.java     # Counselor entity with atomic lead counter
      │    └── allocation/
      │         ├── LeadAllocator.java # Strategy interface for lead assignment
      │         └── RoundRobinAllocator.java # High-concurrency atomic allocator
      └── test/java/com/simplyadmission/core/
           └── allocation/
                └── RoundRobinAllocatorTest.java # 50-thread concurrent stress test
```

---

## 🚀 Running the Tests
Once OpenJDK 21 and Maven are installed:
```bash
mvn test
```
This executes the suite including the `highConcurrencyAllocationTest`, which fires 50 concurrent threads distributing 1,000 incoming leads across active counselors with zero race conditions.
