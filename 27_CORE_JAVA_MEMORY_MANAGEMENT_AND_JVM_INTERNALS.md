# Tutorial 27: Java Memory Management, Garbage Collection & JVM Internals

Java handles memory automatically through Garbage Collection (GC). However, if you do not understand where variables live (Stack vs. Heap) or how objects become unreachable, you will create silent **Memory Leaks** that crash servers during peak admission traffic.

---

## 1. JVM Memory Layout: Stack vs. Heap vs. Metaspace

```
┌────────────────────────────────────────────────────────────────────────┐
│                               JVM MEMORY                               │
├────────────────────────────────┬───────────────────────────────────────┤
│ STACK (Per-Thread)             │ HEAP (Shared by All Threads)          │
├────────────────────────────────┼───────────────────────────────────────┤
│ • Thread-safe & fast.          │ • Stores ALL object instances & arrays│
│ • Stores method call frames.   │ • Managed by Garbage Collector.       │
│ • Holds primitive values       │ • Split into Generational Spaces:     │
│   (`int`, `double`, `boolean`).│   - Young Gen (Eden, S0, S1)          │
│ • Holds object REFERENCE       │   - Old Gen (Tenured)                 │
│   pointers (addresses).        │                                       │
├────────────────────────────────┴───────────────────────────────────────┤
│ METASPACE (Off-Heap Native Memory)                                     │
│ • Stores Class metadata, static methods, bytecode definitions.         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. The Universal Misunderstanding: Is Java Pass-by-Value?

> **"Java is 100% strictly Pass-by-Value. There is NO pass-by-reference in Java."**

### Why Developers Get Confused:
When you pass an object into a method, you are **not** passing the object itself—you are passing a **copy of the reference pointer (memory address)** by value!

```java
public class MemoryDemo {
    public static void modify(Lead lead) {
        lead.setName("Priya"); // Mutates the object on the Heap! (Affects caller)
        lead = new Lead("Rohan"); // Reassigns local pointer copy! (Does NOT affect caller)
    }

    public static void main(String[] args) {
        Lead myLead = new Lead("Aarav");
        modify(myLead);
        System.out.println(myLead.getName()); // Prints "Priya", NEVER "Rohan"!
    }
}
```

---

## 3. How Garbage Collection Works (Reachability Analysis)

The JVM does **not** use naive reference counting. It uses **Root Reachability Analysis**:

```
[ GC ROOTS ] (Active Thread Stacks, Static Variables, JNI References)
     │
     ├──► Object A (Reachable ➔ KEPT ALIVE)
     │        │
     │        └──► Object B (Reachable ➔ KEPT ALIVE)
     │
[ Unreachable Island ]
Object C ◄──► Object D (They reference each other, but NO path from GC Roots!)
                       (➔ COLLECTED AND WIPED FROM RAM!)
```

### The Generational Hypothesis:
1. **Most objects die young**: 95% of objects (like short-lived DTOs or string builders inside a method) become dead within milliseconds.
2. **Eden Space**: New objects are created in **Eden**.
3. **Survivor Spaces (S0 & S1)**: Objects that survive a Minor GC are copied to Survivor spaces and given an "age" counter.
4. **Tenured / Old Generation**: If an object survives multiple GC cycles (e.g., Spring Beans, database connection pools), it is promoted to the **Old Gen**.

---

## 4. Modern Garbage Collectors (G1GC vs. ZGC)

* **G1GC (Garbage-First GC - Default in Java 17/21)**:
  * Divides the heap into hundreds of equal memory regions.
  * Pauses application threads for 10–50ms to clean regions with the most garbage.
* **ZGC (Z Garbage Collector - Low Latency)**:
  * Performs almost all GC work concurrently with application threads.
  * Guarantees pause times under **1 millisecond**, even on multi-terabyte heaps! Ideal for high-frequency financial and admission platforms.

---

## 5. How Memory Leaks Happen in Java (And How to Fix Them)

Even with Garbage Collection, memory leaks occur when **unneeded objects remain attached to a GC Root**.

### Leak 1: Static Collections (The Silent Heap Killer)
```java
// Anti-Pattern: A static list that grows forever and is NEVER cleared!
public class LeadCache {
    private static final List<Lead> CACHE = new ArrayList<>(); // GC ROOT!
    
    public static void add(Lead lead) {
        CACHE.add(lead); // Never removed! Eventually causes OutOfMemoryError.
    }
}
```
* **Fix**: Use a cache with automatic eviction (like **Caffeine Cache** or **Redis**) with time-to-live (TTL) expiry.

### Leak 2: Uncleaned `ThreadLocal` in Thread Pools
* Web servers (Tomcat/Jetty) reuse threads from a pool.
* If you set `TenantContextHolder.set(tenantInfo)` and **forget to call `.remove()` in a `finally` block**, that tenant info stays attached to the thread permanently, leaking memory and cross-contaminating other users' requests!

```java
try {
    TenantContextHolder.set(info);
    processRequest();
} finally {
    TenantContextHolder.clear(); // MANDATORY CLEANUP!
}
```
