# Tutorial 4: Interactive Java Practice Lab (Hands-on Learning)

This lab teaches you Java step-by-step through real-world **SimplyAdmission** challenges. Every exercise includes a real problem, starter code, a test challenge, and explanations.

---

## 🧪 Exercise 1: Conditions & Modern Switch Pattern Matching
*Concept: Making decisions, validating applicant ages, and determining admission eligibility.*

### The Real Problem
A parent wants to enroll their child in DPS. The school has strict age criteria:
* Age < 3: **Ineligible** (Too young for Nursery).
* Age 3 to 4: **Nursery**.
* Age 5 to 6: **Kindergarten (KG)**.
* Age 6 to 16: **Primary & Secondary** (Grade = Age - 5).
* Age > 16: **Senior Secondary (Grades 11–12)**.

### The Code Pattern
```java
public class AdmissionEligibility {

    public static String determineGrade(int age, boolean entranceTestPassed) {
        // 1. Basic if-else validation
        if (age < 3) {
            return "REJECTED: Child must be at least 3 years old.";
        }

        // 2. Modern Java Pattern: Switch Expressions (returns a value directly!)
        return switch (age) {
            case 3, 4 -> "ELIGIBLE: Nursery (No entrance test needed)";
            case 5 -> "ELIGIBLE: Kindergarten";
            case 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16 -> {
                int grade = age - 5;
                if (!entranceTestPassed) {
                    yield "CONDITIONAL: Eligible for Grade " + grade + " pending Entrance Test.";
                }
                yield "ACCEPTED: Confirmed for Grade " + grade;
            }
            default -> "REFERRED: Senior Admissions Desk for Grade 11/12 Stream Selection";
        };
    }

    public static void main(String[] args) {
        System.out.println(determineGrade(2, false)); // REJECTED
        System.out.println(determineGrade(4, false)); // Nursery
        System.out.println(determineGrade(10, false)); // CONDITIONAL: Grade 5
        System.out.println(determineGrade(10, true));  // ACCEPTED: Grade 5
    }
}
```

> 🎯 **Your Practice Challenge**:
> Add a rule: If the student has a sibling already studying in the school (`boolean hasSibling`), they get an automatic waiver on the entrance test for Grades 1 to 5.

---

## 🧪 Exercise 2: Loops & Iterations (Processing Batches of Inquiries)
*Concept: Iterating over collections, `for-each`, `while`, and modern Java Streams.*

### The Real Problem
At 9:00 AM, our webhook received a batch of 5 inquiries. We need to:
1. Loop through each student.
2. Clean up any accidental leading/trailing spaces in their names.
3. Calculate an engagement score.
4. Stop processing if an inquiry has an emergency flag.

### The Code Pattern
```java
import java.util.List;

public class BatchLeadProcessor {

    record Inquiry(String studentName, String phone, int score) {}

    public static void main(String[] args) {
        List<Inquiry> rawBatch = List.of(
            new Inquiry("  Aarav Sharma ", "+919876543210", 45),
            new Inquiry("Priya Singh", "+919123456780", 85),
            new Inquiry(" Rohan Verma ", "+919988776655", 10),
            new Inquiry("Sneha Patel", "+919811223344", 95)
        );

        System.out.println("=== 1. TRADITIONAL FOR-EACH LOOP ===");
        for (Inquiry inq : rawBatch) {
            String cleanName = inq.studentName().trim();
            if (inq.score() >= 80) {
                System.out.println("🔥 High Priority Lead: " + cleanName + " (Score: " + inq.score() + ")");
            } else {
                System.out.println("Standard Lead: " + cleanName);
            }
        }

        System.out.println("\n=== 2. MODERN JAVA STREAM FILTER ===");
        // Functional pipeline: filter -> map -> print
        rawBatch.stream()
            .filter(inq -> inq.score() >= 80)
            .map(inq -> inq.studentName().trim().toUpperCase())
            .forEach(name -> System.out.println("SMS Sent to: " + name));
    }
}
```

---

## 🧪 Exercise 3: Lists, Sets & Maps (Deduplication & Grouping)
*Concept: When to use `List` vs `Set` vs `Map` in backend systems.*

### The Real Problem
* **List**: We need an ordered list of students who submitted forms (`ArrayList`).
* **Set**: Parents sometimes click submit twice. We must deduplicate by phone number (`HashSet`).
* **Map**: Counselors need quick lookups: Given a `CounselorId`, get all their assigned students (`HashMap<String, List<Lead>>`).

### The Code Pattern
```java
import java.util.*;

public class CollectionMastery {

    public static void main(String[] args) {
        // 1. SET: Automatic Deduplication
        Set<String> uniqueParentPhones = new HashSet<>();
        uniqueParentPhones.add("+919876543210");
        uniqueParentPhones.add("+919876543210"); // Duplicate! Won't be added twice.
        uniqueParentPhones.add("+919123456789");

        System.out.println("Unique phone numbers count: " + uniqueParentPhones.size()); // 2

        // 2. MAP: Counselor to Students Mapping (Key-Value)
        Map<String, List<String>> counselorBucket = new HashMap<>();

        // Helper method to assign student to a counselor bucket
        assignStudent(counselorBucket, "Counselor_Pooja", "Aarav Sharma (Grade 5)");
        assignStudent(counselorBucket, "Counselor_Pooja", "Rohan Verma (Grade 3)");
        assignStudent(counselorBucket, "Counselor_Rahul", "Priya Singh (Grade 1)");

        System.out.println("\nCounselor Workloads:");
        for (Map.Entry<String, List<String>> entry : counselorBucket.entrySet()) {
            System.out.println(entry.getKey() + " has " + entry.getValue().size() + " leads: " + entry.getValue());
        }
    }

    private static void assignStudent(Map<String, List<String>> map, String counselor, String student) {
        // computeIfAbsent: If counselor isn't in the map yet, create a new ArrayList for them!
        map.computeIfAbsent(counselor, k -> new ArrayList<>()).add(student);
    }
}
```

---

## 🧪 Exercise 4: JSON Handling Without External Libraries
*Concept: JSON is the universal language of webhooks, browsers, and mobile apps.*

### The Real Problem
Meta sends an ad inquiry formatted as a JSON string:
`{"student": "Aarav", "phone": "+919876543210", "grade": "Grade 5"}`.
We need to extract the values and create a JSON response to send back to the browser.

### The Code Pattern (Pure Java 21)
```java
public class SimpleJsonParser {

    public static void main(String[] args) {
        String jsonPayload = "{\"student\": \"Aarav\", \"phone\": \"+919876543210\", \"grade\": \"Grade 5\"}";

        // Extracting fields using simple string manipulation (Mental model of what Jackson/Gson does)
        String student = extractJsonValue(jsonPayload, "student");
        String phone = extractJsonValue(jsonPayload, "phone");
        String grade = extractJsonValue(jsonPayload, "grade");

        System.out.println("Parsed Student: " + student);
        System.out.println("Parsed Phone: " + phone);
        System.out.println("Parsed Grade: " + grade);

        // Building an outgoing JSON response
        String jsonResponse = buildJsonResponse("SUCCESS", "Inquiry registered for " + student);
        System.out.println("\nResponse to Frontend:\n" + jsonResponse);
    }

    public static String extractJsonValue(String json, String key) {
        String pattern = "\"" + key + "\":\\s*\"";
        int start = json.indexOf(pattern);
        if (start == -1) return null;
        start += pattern.length();
        int end = json.indexOf("\"", start);
        return json.substring(start, end);
    }

    public static String buildJsonResponse(String status, String message) {
        return "{\n  \"status\": \"" + status + "\",\n  \"message\": \"" + message + "\"\n}";
    }
}
```

---

## 🧪 Exercise 5: Table Handling & File Data Saving (Local Persistence)
*Concept: How databases work under the hood. Reading & writing rows to disk.*

### The Real Problem
Before connecting PostgreSQL, you must understand how structured data is written to a file and read back as a table.

### The Code Pattern
```java
import java.io.*;
import java.nio.file.*;
import java.util.*;

public class LocalDataStore {

    private static final Path DB_FILE = Paths.get("admissions_table.csv");

    public record StudentRecord(String id, String name, String phone, String grade, String status) {
        public String toCsvRow() {
            return String.join(",", id, name, phone, grade, status);
        }

        public static StudentRecord fromCsvRow(String row) {
            String[] p = row.split(",");
            return new StudentRecord(p[0], p[1], p[2], p[3], p[4]);
        }
    }

    public static void main(String[] args) throws IOException {
        // 1. Initialize file with table headers if it doesn't exist
        if (!Files.exists(DB_FILE)) {
            Files.writeString(DB_FILE, "ID,NAME,PHONE,GRADE,STATUS\n");
            System.out.println("Created table admissions_table.csv");
        }

        // 2. INSERT: Save records to disk
        saveStudent(new StudentRecord("ADM-101", "Aarav Sharma", "+919876543210", "Grade 5", "NEW"));
        saveStudent(new StudentRecord("ADM-102", "Priya Singh", "+919123456789", "Grade 2", "CONTACTED"));

        // 3. SELECT: Read all rows and print as a formatted table
        List<StudentRecord> allStudents = readAllStudents();

        System.out.println("\n------------------------------------------------------------");
        System.out.printf("%-10s | %-16s | %-15s | %-10s | %-10s%n", "ID", "NAME", "PHONE", "GRADE", "STATUS");
        System.out.println("------------------------------------------------------------");
        for (StudentRecord s : allStudents) {
            System.out.printf("%-10s | %-16s | %-15s | %-10s | %-10s%n", s.id(), s.name(), s.phone(), s.grade(), s.status());
        }
        System.out.println("------------------------------------------------------------");
    }

    public static void saveStudent(StudentRecord student) throws IOException {
        Files.writeString(DB_FILE, student.toCsvRow() + "\n", StandardOpenOption.CREATE, StandardOpenOption.APPEND);
        System.out.println("Saved student: " + student.name() + " to disk!");
    }

    public static List<StudentRecord> readAllStudents() throws IOException {
        List<String> lines = Files.readAllLines(DB_FILE);
        List<StudentRecord> list = new ArrayList<>();
        // Skip header at index 0
        for (int i = 1; i < lines.size(); i++) {
            if (!lines.get(i).isBlank()) {
                list.add(StudentRecord.fromCsvRow(lines.get(i)));
            }
        }
        return list;
    }
}
```
