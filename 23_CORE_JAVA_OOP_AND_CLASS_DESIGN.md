# Tutorial 23: Core Java OOP & Class Design Mastery

Object-Oriented Programming (OOP) is the foundation of Java. However, in enterprise backend systems, OOP is not about academic examples like `Animal -> Dog`. It is about writing clean, maintainable, decoupled domain models.

---

## 1. The 4 Pillars of OOP in Enterprise Systems

### 1. Encapsulation (Hiding Internal State)
* **Definition**: Bundling data with the methods that operate on that data and restricting direct access to object internals.
* **Production Rule**: Fields must **always** be `private`. Mutating state should only happen through methods that enforce domain rules:
```java
public class Lead {
    private int engagementScore; // Private: cannot be set directly from outside

    public void recordCallInteraction(int callDurationSeconds) {
        if (callDurationSeconds > 30) {
            // Business Rule: Only increment score if call lasted > 30s
            this.engagementScore = Math.min(100, this.engagementScore + 10);
        }
    }
}
```

### 2. Abstraction (Hiding Complexity Behind Contracts)
* **Definition**: Showing only essential features to the outside world while hiding the messy implementation.
* **Production Rule**: Program to **interfaces**, not concrete classes:
```java
// Other services don't care IF it's SendGrid, AWS SES, or Mailgun
public interface EmailSender {
    void send(String to, String subject, String body);
}
```

### 3. Inheritance vs. Composition (The #1 Junior Dev Mistake)
* **The Trap**: Overusing `extends` creates brittle class hierarchies where changing a parent class breaks 10 children.
* **The Rule: Favor Composition Over Inheritance**:
  * *Inheritance (IS-A)*: A `StudentApplication` IS AN `Application`.
  * *Composition (HAS-A)*: A `StudentApplication` HAS A `PaymentMethod` and HAS A `List<Document>`.

### 4. Polymorphism (One Interface, Multiple Behaviors)
* **Compile-Time (Overloading)**: Same method name, different parameter signatures (`allocate(lead)`, `allocate(lead, priority)`).
* **Runtime (Overriding)**: A subclass or interface implementation provides its own behavior (`RoundRobinAllocator` vs `ScoreWeightedAllocator`).

---

## 2. Abstract Classes vs. Interfaces (Modern Java 21)

| Feature | `interface` | `abstract class` |
| :--- | :--- | :--- |
| **Primary Purpose** | Defines a **Contract / Capability** (*what* it can do). | Defines an **Incomplete Base Class** (*what* it is). |
| **Multiple Inheritance** | A class can implement **multiple** interfaces. | A class can extend **only one** class. |
| **State / Fields** | Only `public static final` constants. | Can have instance variables (`private int score;`). |
| **Constructors** | **Cannot** have constructors. | **Can** have constructors for subclasses. |
| **Method Implementation**| Can have `default` and `static` methods (Java 8+), and `private` helper methods (Java 9+). | Can have fully implemented methods and abstract methods. |

---

## 3. The Sacred `equals()` and `hashCode()` Contract

Violating the `equals()` and `hashCode()` contract is one of the most common causes of silent data corruption in Java.

### The Contract Rules:
1. If two objects are equal according to `equals()`, their `hashCode()` **MUST be identical**.
2. If two objects have the same `hashCode()`, they are **not necessarily equal** (Hash Collision).
3. If you override `equals()`, you **MUST** override `hashCode()`.

### The Disaster If You Violate It:
```java
public class Counselor {
    private String id; // Forgot to override equals and hashCode!
    public Counselor(String id) { this.id = id; }
}

// In your CRM code:
Set<Counselor> activeCounselors = new HashSet<>();
activeCounselors.add(new Counselor("c-101"));

// Checking if counselor exists:
boolean exists = activeCounselors.contains(new Counselor("c-101"));
System.out.println(exists); // PRINTS FALSE! Why? Memory addresses differed!
```

### The Correct Implementation (Modern Java):
```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Counselor that)) return false;
    return Objects.equals(id, that.id) && Objects.equals(tenantId, that.tenantId);
}

@Override
public int hashCode() {
    return Objects.hash(id, tenantId);
}
```

---

## 4. `Comparable` vs. `Comparator` (Sorting Domain Entities)

* **`Comparable<T>` (Natural Ordering)**: Implemented on the entity itself via `compareTo()`.
* **`Comparator<T>` (Custom / Multiple Sort Orders)**: Separate strategy classes or lambdas.

```java
// 1. Natural Sort: Applications sorted chronologically by submission date
public class Application implements Comparable<Application> {
    private Instant createdAt;

    @Override
    public int compareTo(Application other) {
        return this.createdAt.compareTo(other.createdAt);
    }
}

// 2. Custom Sorts via Comparator (Modern Java Lambdas)
List<Lead> leads = getLeads();

// Sort by Engagement Score descending (highest score first)
leads.sort(Comparator.comparingInt(Lead::engagementScore).reversed());

// Multi-level sorting: Sort by Grade applied, then by Student Name alphabetically
leads.sort(
    Comparator.comparing(Lead::gradeApplied)
              .thenComparing(Lead::studentName)
);
```
