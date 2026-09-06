# Tutorial 25: Functional Java, Lambdas & Streams API Mastery

Java 8 transformed the language from purely object-oriented to a hybrid **Object-Functional** powerhouse. In modern enterprise Spring Boot development, more than 50% of your business data processing will be written using **Lambdas, Functional Interfaces, and the Streams API**.

---

## 1. The 4 Core Functional Interfaces (`java.util.function`)

A **Functional Interface** has exactly ONE abstract method (Single Abstract Method - SAM). You must know these four by heart:

| Interface | Method Signature | Purpose | Real CRM Example |
| :--- | :--- | :--- | :--- |
| **`Predicate<T>`** | `boolean test(T t)` | **Filter & Validate** | `lead -> lead.engagementScore() > 80` |
| **`Function<T, R>`** | `R apply(T t)` | **Transform / Map** | `lead -> lead.parentPhone()` (Lead ➔ String) |
| **`Consumer<T>`** | `void accept(T t)` | **Consume / Action** | `lead -> whatsAppService.send(lead)` |
| **`Supplier<T>`** | `T get()` | **Produce / Factory** | `() -> new ResourceNotFoundException("Not found")` |

---

## 2. Lambdas & Method References

A lambda expression is an anonymous function (a function without a name).

```java
// 1. Traditional Anonymous Class (Old & Verbose)
Collections.sort(leads, new Comparator<Lead>() {
    @Override
    public int compare(Lead l1, Lead l2) {
        return l1.studentName().compareTo(l2.studentName());
    }
});

// 2. Lambda Expression (Clean)
leads.sort((l1, l2) -> l1.studentName().compareTo(l2.studentName()));

// 3. Method Reference (Cleanest)
leads.sort(Comparator.comparing(Lead::studentName));
```

### 3 Types of Method References:
* **Static method**: `String::valueOf` ➔ `x -> String.valueOf(x)`
* **Instance method of an object**: `System.out::println` ➔ `x -> System.out.println(x)`
* **Instance method of an arbitrary object**: `String::toUpperCase` ➔ `str -> str.toUpperCase()`

---

## 3. The Streams API: Collections vs. Streams

* **Collection**: An in-memory data structure holding all its elements (e.g., an `ArrayList` of 10,000 students).
* **Stream**: A computational pipeline that transforms elements **on-demand (lazily)** without modifying the source collection!

```
Source Collection ➔ [filter] ➔ [map] ➔ [sorted] ➔ [collect] ➔ Result
                    └───────────┬─────────────┘    └────┬────┘
                         Intermediate Operations      Terminal Operation
                            (Lazy Evaluation)        (Triggers Execution)
```

### Example: Processing Top Admission Candidates
```java
List<String> topCandidatePhones = allLeads.stream()
    // 1. Intermediate: Only Nursery applicants
    .filter(lead -> "Nursery".equalsIgnoreCase(lead.gradeApplied()))
    // 2. Intermediate: Only high intent
    .filter(lead -> lead.engagementScore() >= 80)
    // 3. Intermediate: Transform Lead object -> Phone String
    .map(Lead::parentPhone)
    // 4. Intermediate: Remove duplicate numbers
    .distinct()
    // 5. Intermediate: Limit to first 50
    .limit(50)
    // 6. Terminal: Package into a List
    .toList();
```

---

## 4. `map()` vs. `flatMap()`

* **`map()` (One-to-One)**: Transforms each element into another object.
  `Lead ➔ String (lead.studentName())`
* **`flatMap()` (One-to-Many / Flattening)**: Transforms each element into a stream, then flattens all nested streams into a single flat stream:
```java
class SchoolBranch {
    List<String> counselorEmails;
}

List<SchoolBranch> branches = getBranches();

// map() produces a List of Lists: [[a@dps, b@dps], [c@dps]]
// flatMap() flattens it into a single list: [a@dps, b@dps, c@dps]
List<String> allEmails = branches.stream()
    .flatMap(branch -> branch.getCounselorEmails().stream())
    .distinct()
    .toList();
```

---

## 5. Advanced Collectors (`Collectors.groupingBy`)

In CRM reporting, `Collectors.groupingBy()` is your best friend:

### A. Grouping Leads by Grade:
```java
Map<String, List<Lead>> leadsByGrade = allLeads.stream()
    .collect(Collectors.groupingBy(Lead::gradeApplied));

// Result: {"Grade 1": [Lead1, Lead2], "Grade 5": [Lead3]}
```

### B. Counting Leads per Counselor (Aggregations):
```java
Map<String, Long> leadsPerCounselor = allLeads.stream()
    .collect(Collectors.groupingBy(
        Lead::assignedCounselorId, 
        Collectors.counting()
    ));

// Result: {"c-101": 42, "c-102": 38}
```

---

## 6. Mastering `Optional<T>` (Eliminating NullPointerExceptions)

An `Optional<T>` is a single-element container that either contains a non-null value or is empty.

### The 3 Golden Rules of `Optional`:
1. **Never call `.get()` without checking `.isPresent()`**: Calling `.get()` on an empty optional throws `NoSuchElementException` (just as bad as a NPE!).
2. **Use `.orElseGet()` instead of `.orElse()`**:
   * `orElse(expensiveCalculation())` executes the method **every time**, even if the optional has a value!
   * `orElseGet(() -> expensiveCalculation())` is lazy and executes **only if empty**.
3. **Use `.map()` and `.flatMap()` to chain operations safely**:

```java
// Anti-Pattern (Ugly null checks)
Lead lead = findLead("101");
if (lead != null) {
    Counselor c = lead.getCounselor();
    if (c != null) {
        return c.getEmail();
    }
}
return "unassigned@dps.edu";

// Modern Java Optional Chain (Clean & Defensive)
return findLeadOptional("101")
    .map(Lead::getCounselor)
    .map(Counselor::getEmail)
    .orElse("unassigned@dps.edu");
```
