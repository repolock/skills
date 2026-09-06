# Tutorial 26: Java Concurrency & Multithreading Mastery

Concurrency is what separates junior coders from senior backend engineers. When 10,000 parents submit forms or 100 counselors query dashboards at the same second, your code must handle shared memory safely without deadlocking or producing corrupted data.

---

## 1. Concurrency vs. Parallelism

* **Concurrency**: Managing multiple tasks at the same time (interleaving progress on a single CPU core).
* **Parallelism**: Executing multiple calculations at the exact same physical nanosecond (across multiple CPU cores).

---

## 2. The 3 Core Multithreading Hazards

### 1. Race Conditions (The "Lost Update")
* Occurs when two threads read and modify shared data simultaneously:
  `Thread A reads count (5) ➔ Thread B reads count (5) ➔ Thread A writes 6 ➔ Thread B writes 6.`
  Two increments happened, but the counter only increased by 1!

### 2. Visibility Issues & CPU Caches
* Modern CPUs have L1, L2, and L3 caches. If Thread A modifies a variable on Core 1, Thread B running on Core 2 might read stale cached data from its local cache unless memory synchronization is enforced!

### 3. Deadlocks
* Thread 1 holds Lock A and waits for Lock B.
* Thread 2 holds Lock B and waits for Lock A.
* Both threads freeze permanently.

---

## 3. The Synchronization Toolkit

### A. `volatile` (Visibility Guarantee, NOT Atomicity)
* **What it does**: Forces reads and writes to go directly to main memory (RAM), bypassing CPU caches.
* **Limitation**: It guarantees visibility, but **does NOT make compound operations atomic**. `volatile int count; count++` is still vulnerable to race conditions!
* **Use Case**: Status flags, e.g., `private volatile boolean serverRunning = true;`.

### B. Atomic Primitives (Hardware-Level CAS)
* Primitives like `AtomicInteger`, `AtomicBoolean`, and `AtomicReference` use CPU-level **Compare-And-Swap (CAS)** instructions.
* They are completely lock-free and 10x faster than `synchronized`:
```java
private final AtomicInteger assignedCount = new AtomicInteger(0);

// Thread-safe increment without locks!
int newCount = assignedCount.incrementAndGet();
```

### C. `synchronized` & Explicit Locks (`ReentrantLock`)
* Places a mutual exclusion lock around a code block:
```java
// Synchronized block on an intrinsic monitor
synchronized (lockObject) {
    // Only ONE thread can execute inside this block at any time
}

// ReentrantLock (Advanced: supports timeouts and fairness)
ReentrantLock lock = new ReentrantLock();
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // Critical section
    } finally {
        lock.unlock(); // Always unlock in finally block!
    }
}
```

---

## 4. Thread Pools & `ExecutorService` (Never Create Raw Threads!)

Creating a platform thread (`new Thread()`) costs ~1MB of memory and requires an expensive operating system context switch. If 5,000 requests hit your server, creating 5,000 threads will trigger an `OutOfMemoryError`.

### Production Tuning of `ThreadPoolExecutor`:
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                      // Core pool size: 10 threads always kept alive
    50,                      // Maximum pool size: Can expand to 50 under load
    60L, TimeUnit.SECONDS,   // Keep-alive time for idle extra threads
    new ArrayBlockingQueue<>(500), // Queue holding up to 500 waiting tasks
    new ThreadPoolExecutor.CallerRunsPolicy() // Backpressure: If queue is full, calling thread runs it!
);
```

---

## 5. Modern Asynchronous Orchestration (`CompletableFuture`)

In microservices, you often need to fetch data from 3 external services simultaneously (e.g., School ERP, Payment Gateway, and SMS Status) and merge them:

```java
// 1. Asynchronously fetch student details
CompletableFuture<Student> studentFuture = CompletableFuture.supplyAsync(() -> erpClient.getStudent(id));

// 2. Asynchronously fetch payment history
CompletableFuture<PaymentHistory> paymentFuture = CompletableFuture.supplyAsync(() -> paymentClient.getHistory(id));

// 3. Combine both results once both finish (Non-blocking!)
CompletableFuture<AdmissionDossier> dossierFuture = studentFuture.thenCombine(paymentFuture, (student, payments) -> {
    return new AdmissionDossier(student, payments);
}).exceptionally(ex -> {
    log.error("Failed to compile dossier", ex);
    return AdmissionDossier.empty();
});

AdmissionDossier dossier = dossierFuture.join(); // Awaits completion
```

---

## 6. Java 21 Superpower: Virtual Threads (Project Loom)

* **Platform Thread**: 1 Java Thread = 1 Operating System Kernel Thread (~1MB RAM). Limit: ~2,000 threads per server.
* **Virtual Thread**: Lightweight JVM-managed thread (~1KB RAM). Limit: **Millions of threads per server!**

```java
// Java 21: One Virtual Thread per Task Executor
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        final int id = i;
        executor.submit(() -> {
            // Blocking network call to WhatsApp API
            Thread.sleep(1000); 
            System.out.println("Notification sent: " + id);
            return id;
        });
    }
} // 100,000 concurrent network tasks run effortlessly with negligible RAM!
```
When a virtual thread executes a blocking I/O operation (`Thread.sleep`, database query, HTTP call), the JVM automatically unmounts it from the underlying carrier thread and lets other virtual threads run!
