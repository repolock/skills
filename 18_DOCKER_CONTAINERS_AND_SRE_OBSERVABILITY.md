# Tutorial 18: Docker Containers, CI/CD & SRE Observability (Prometheus & Grafana)

Building code that works on your laptop is only 50% of the job. A production-ready backend engineer must know how to:
1. **Containerize** the application with Docker so it runs identically on any cloud server.
2. **Monitor** live application health with Prometheus and Grafana before servers crash.

---

## 1. Multi-Stage Dockerfile for Spring Boot (Production Standard)

A naive `Dockerfile` creates bloated 800MB images containing Maven, compilers, and source code.
A **Multi-Stage Dockerfile** compiles the code in Stage 1, then copies **only the lightweight JAR** into a secure, hardened runtime image in Stage 2 (resulting in an image under 150MB!).

File: `Dockerfile`
```dockerfile
# ==========================================
# STAGE 1: Build & Package the Application
# ==========================================
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

# Copy dependency configs first (layer caching optimization)
COPY pom.xml .
COPY src ./src

# Compile and package without running tests (tests run in CI/CD)
RUN ./mvnw clean package -DskipTests

# ==========================================
# STAGE 2: Lightweight Production Runtime
# ==========================================
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Security: Run as non-root user (prevents container takeover)
RUN addgroup -S simplygroup && adduser -S simplyuser -G simplygroup
USER simplyuser

# Copy compiled JAR from Stage 1
COPY --from=builder /app/target/simplyadmission-core-1.0.0.jar app.jar

# Expose HTTP Port
EXPOSE 8080

# Production JVM Flags for Container Memory Awareness
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:+UseG1GC", \
  "-jar", "app.jar"]
```

---

## 2. The Complete Local Production Environment (`docker-compose.yml`)

With one single command (`docker compose up -d`), you can launch the entire SimplyAdmission ecosystem locally:

File: `docker-compose.yml`
```yaml
version: '3.8'

services:
  # 1. PostgreSQL 16 Database
  postgres:
    image: postgres:16-alpine
    container_name: simplyadmission-db
    environment:
      POSTGRES_DB: simplyadmission_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: mysecretpassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  # 2. Redis 7 Cache & Distributed Locks
  redis:
    image: redis:7-alpine
    container_name: simplyadmission-redis
    ports:
      - "6379:6379"

  # 3. SimplyAdmission Java Backend
  api:
    build: .
    container_name: simplyadmission-api
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/simplyadmission_db
      SPRING_DATA_REDIS_HOST: redis

  # 4. Prometheus Metrics Scraper
  prometheus:
    image: prom/prometheus:latest
    container_name: simplyadmission-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  # 5. Grafana Visual Telemetry Dashboard
  grafana:
    image: grafana/grafana:latest
    container_name: simplyadmission-grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

volumes:
  postgres_data:
```

---

## 3. Exposing Metrics with Spring Boot Actuator & Micrometer

Spring Boot includes **Actuator**, a built-in module that tracks memory, threads, and HTTP latency.

Add to `application.yml`:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: always
    prometheus:
      enabled: true
```

When you visit `http://localhost:8080/actuator/prometheus`, Spring Boot emits real-time telemetry:
```
# HELP jvm_memory_used_bytes The amount of used memory
jvm_memory_used_bytes{area="heap",id="G1 Eden Space"} 4.52984832E8
# HELP http_server_requests_seconds
http_server_requests_seconds_count{method="POST",uri="/api/v1/leads"} 8492
http_server_requests_seconds_max{method="POST",uri="/api/v1/leads"} 0.042
```

---

## 4. Setting Up Prometheus to Scrape SimplyAdmission

File: `prometheus.yml`
```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'simplyadmission-backend'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['api:8080']
```

---

## 5. Grafana Dashboard: The 4 Golden Signals of SRE

In Grafana (`http://localhost:3000`), Site Reliability Engineers monitor the **4 Golden Signals**:

```
┌────────────────────────────────────────────────────────────────────────┐
│                   SIMPLYADMISSION GRAFANA DASHBOARD                    │
├───────────────────────────────────┬────────────────────────────────────┤
│ 1. LATENCY (P99 Response Time)    │ 2. TRAFFIC (Requests Per Second)   │
│ Target: < 100ms                   │ Current: 450 req/sec               │
│ [ Graph: 25ms ─── 32ms ─── 40ms ] │ [ Graph: Steady Peak ]             │
├───────────────────────────────────┼────────────────────────────────────┤
│ 3. ERRORS (5xx Error Rate)        │ 4. SATURATION (HikariCP / JVM RAM) │
│ Current: 0.01% (Normal)           │ DB Pool: 6 / 20 active             │
│ [ Alert: Threshold > 1% ]         │ Heap: 35% utilized (Healthy)       │
└───────────────────────────────────┴────────────────────────────────────┘
```

### Why This Matters in SimplyAdmission:
On **Board Exam Result Day**, 50,000 parents check their admission cut-offs simultaneously. 
* Without Grafana, your server crashes silently in the dark.
* With Grafana, you see connection pool saturation climbing at 9:02 AM, and autoscaling triggers two additional container instances automatically, keeping the portal live with 100% uptime!
