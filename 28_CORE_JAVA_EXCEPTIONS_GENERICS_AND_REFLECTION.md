# Tutorial 28: Exceptions, Generics & Reflection Mastery

This tutorial covers the advanced structural mechanisms of Java: handling failures cleanly through **Exception Hierarchies**, writing reusable type-safe algorithms with **Generics & Wildcards**, and understanding how Spring Boot works under the hood via **Reflection & Custom Annotations**.

---

## 1. Exception Architecture & Defensive Design

```
                     ┌──────────────────┐
                     │    Throwable     │
                     └────────┬─────────┘
            ┌─────────────────┴─────────────────┐
            ▼                                   ▼
     ┌─────────────┐                     ┌─────────────┐
     │    Error    │ (Unrecoverable)     │  Exception  │
     │ (OOM, Stack │                     └──────┬──────┘
     │  Overflow)  │            ┌───────────────┴───────────────┐
     └─────────────┘            ▼                               ▼
                         Checked Exceptions           Unchecked Exceptions
                         (Forced by compiler)         (Subclasses of RuntimeException)
                         • IOException                • NullPointerException
                         • SQLException               • IllegalArgumentException
                                                      • Business Exceptions!
```

### Why Modern Frameworks Avoid Checked Exceptions:
Checked exceptions clutter code with endless boilerplate `throws IOException` cascades. Modern architectures wrap low-level failures into **Unchecked Domain Exceptions**:

```java
// 1. Base CRM Domain Exception
public abstract class SimplyAdmissionException extends RuntimeException {
    private final String errorCode;

    public SimplyAdmissionException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}

// 2. Concrete Business Failure
public class LeadAlreadyAssignedException extends SimplyAdmissionException {
    public LeadAlreadyAssignedException(String leadId, String counselorId) {
        super("Lead " + leadId + " is already assigned to counselor " + counselorId, "LEAD_ALREADY_ASSIGNED");
    }
}
```

---

## 2. Deterministic Cleanup: `try-with-resources`

Before Java 7, unclosed files and database connections leaked memory. Today, any class implementing **`AutoCloseable`** cleans up automatically:

```java
// Sockets, Database Connections, and File Streams are closed automatically!
try (InputStream is = Files.newInputStream(path);
     Connection conn = dataSource.getConnection()) {
    
    // Perform operations
} catch (IOException | SQLException e) {
    log.error("Resource operation failed", e);
} // Both 'is' and 'conn' are 100% guaranteed to be closed, even if an exception occurs!
```

---

## 3. Generics & The PECS Rule (Producer Extends, Consumer Super)

Generics enforce type-safety at compile time. At runtime, the JVM performs **Type Erasure** (replacing generic types with `Object` for backward compatibility).

### The Wildcard Puzzle:
* **`? extends T` (Upper Bounded)**: Use when you only **READ** from a structure (Producer).
* **`? super T` (Lower Bounded)**: Use when you only **WRITE** into a structure (Consumer).

> **Rule: PECS (Producer Extends, Consumer Super)**

```java
public class GenericCollectionUtils {

    // PRODUCER: We only READ numbers from src (so use ? extends Number)
    // CONSUMER: We WRITE numbers into dest (so use ? super Number)
    public static void copyNumbers(List<? extends Number> src, List<? super Number> dest) {
        for (Number n : src) {
            dest.add(n); // Safe to write into a supertype!
        }
    }
}
```

---

## 4. Custom Annotations & How Reflection Powers Spring Boot

Junior developers think Spring Boot is "magic." It is not magic—it is **Java Reflection**!

When you put `@Autowired` or `@RestController` on a class, Spring inspects your class metadata at runtime using Reflection.

### Building Your Own Custom Audit Annotation:
```java
// Step 1: Define Custom Annotation
@Retention(RetentionPolicy.RUNTIME) // Keep metadata alive in memory at runtime!
@Target(ElementType.METHOD)         // Can only be placed on methods
public @interface AuditAction {
    String description();
}

// Step 2: Use It on Business Logic
public class LeadService {
    @AuditAction(description = "Counselor reassigned student lead")
    public void reassignLead(String leadId, String counselorId) {
        // Business logic
    }
}

// Step 3: How the Framework Reads It (Reflection Engine)
public class ReflectionInspector {
    public static void inspect(Object serviceInstance) throws Exception {
        Method[] methods = serviceInstance.getClass().getDeclaredMethods();

        for (Method method : methods) {
            if (method.isAnnotationPresent(AuditAction.class)) {
                AuditAction audit = method.getAnnotation(AuditAction.class);
                System.out.println("Discovered audited method: " + method.getName() + " -> " + audit.description());
            }
        }
    }
}
```
Through Reflection, frameworks can instantiate objects, inject private fields (`field.setAccessible(true)`), and wrap methods with database transaction interceptors.
