# Tutorial 9: PostgreSQL Integration Guide for SimplyAdmission

PostgreSQL 16 is the relational database powerhouse for SimplyAdmission. This guide covers how Java connects to PostgreSQL, executes queries, manages connection pools, handles dynamic `JSONB` school forms, and runs versioned database migrations.

---

## 1. The Architecture of Java ➔ PostgreSQL Connection

```
┌────────────────────────────────────────────────────────────────────────┐
│                        JAVA BACKEND APPLICATION                        │
│                                                                        │
│   Java Code (Entities / DTOs)                                          │
│        ▲                                                               │
│        ▼                                                               │
│   Hibernate / JPA (Object-Relational Mapping: Java Object ◄► SQL Row) │
│        ▲                                                               │
│        ▼                                                               │
│   HikariCP (Connection Pool: 10 pre-opened persistent TCP sockets)     │
│        ▲                                                               │
│        ▼                                                               │
│   PostgreSQL JDBC Driver (Translates SQL into PostgreSQL Wire Protocol)│
└────────┬───────────────────────────────────────────────────────────────┘
         │ (TCP Socket: localhost:5432)
         ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        POSTGRESQL 16 DATABASE                          │
│                                                                        │
│   Tables: tenants, campuses, leads, applications, payments             │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Step 1: Configuration (`application.yml`)

Spring Boot needs to know where PostgreSQL lives and how to authenticate:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/simplyadmission_db?sslmode=disable
    username: postgres
    password: mysecurepassword
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 20       # Max 20 concurrent queries
      minimum-idle: 5             # Keep at least 5 connections warm
      idle-timeout: 300000        # 5 minutes
      connection-timeout: 20000   # Wait max 20 seconds before failing
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
    hibernate:
      ddl-auto: validate          # NEVER use 'update' or 'create' in production!
    show-sql: false               # Keep false in prod for performance
    properties:
      hibernate.format_sql: true
```

---

## 3. Step 2: Database Versioning with Flyway

In production, **we never manually create tables via pgAdmin**. We write versioned SQL scripts. When the Java app starts up, Flyway runs these scripts automatically:

File: `src/main/resources/db/migration/V1__create_crm_tables.sql`
```sql
-- 1. Tenants & Campuses
CREATE TABLE tenants (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE campuses (
    id VARCHAR(50) PRIMARY KEY,
    tenant_id VARCHAR(50) NOT NULL REFERENCES tenants(id),
    name VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL
);

-- 2. Leads Table
CREATE TABLE leads (
    id VARCHAR(50) PRIMARY KEY,
    tenant_id VARCHAR(50) NOT NULL REFERENCES tenants(id),
    campus_id VARCHAR(50) NOT NULL REFERENCES campuses(id),
    student_name VARCHAR(255) NOT NULL,
    parent_phone VARCHAR(20) NOT NULL,
    email VARCHAR(255) NOT NULL,
    stage VARCHAR(50) NOT NULL DEFAULT 'NEW',
    counselor_id VARCHAR(50),
    engagement_score INT DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 3. Composite Index for Counselor Dashboard Queries
CREATE INDEX idx_leads_tenant_campus_stage ON leads (tenant_id, campus_id, stage);
```

---

## 4. Step 3: Handling Custom School Forms with `JSONB`

Different schools ask different questions (e.g., "Bus Route", "Previous School Name", "Sibling Roll Number"). Instead of creating 50 empty columns, we use PostgreSQL **`JSONB`**:

File: `src/main/resources/db/migration/V2__create_applications_with_jsonb.sql`
```sql
CREATE TABLE applications (
    id VARCHAR(50) PRIMARY KEY,
    lead_id VARCHAR(50) NOT NULL REFERENCES leads(id),
    grade_applied VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'DRAFT',
    custom_data JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- GIN (Generalized Inverted Index) makes searches inside JSONB sub-millisecond fast!
CREATE INDEX idx_applications_custom_data ON applications USING GIN (custom_data);
```

### Querying JSONB in SQL:
```sql
-- Find all applicants where bus transportation is requested
SELECT id, grade_applied, custom_data->>'bus_stop' AS stop_name
FROM applications 
WHERE custom_data @> '{"bus_required": true}';
```

---

## 5. Step 4: The Spring Data JPA Entity in Java

```java
package com.simplyadmission.core.entity;

import jakarta.persistence.*;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;
import java.time.Instant;
import java.util.Map;

@Entity
@Table(name = "applications")
public class ApplicationEntity {

    @Id
    private String id;

    @Column(name = "lead_id", nullable = false)
    private String leadId;

    @Column(name = "grade_applied", nullable = false)
    private String gradeApplied;

    @Column(name = "status", nullable = false)
    private String status;

    // Hibernate 6 handles JSONB mapping natively!
    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "custom_data", columnDefinition = "jsonb")
    private Map<String, Object> customData;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    // Standard Getters & Setters
}
```

---

## 6. Step 5: The Repository Layer (Zero SQL Boilerplate)

```java
package com.simplyadmission.core.repository;

import com.simplyadmission.core.entity.ApplicationEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface ApplicationRepository extends JpaRepository<ApplicationEntity, String> {

    // Spring Data JPA auto-generates the SQL query behind the scenes!
    List<ApplicationEntity> findByGradeAppliedAndStatus(String grade, String status);

    // Native PostgreSQL JSONB Query
    @Query(value = "SELECT * FROM applications WHERE custom_data @> cast(:jsonCriteria as jsonb)", nativeQuery = true)
    List<ApplicationEntity> findByCustomJsonField(@Param("jsonCriteria") String jsonCriteria);
}
```
