# Tutorial 24: Java Collections & Data Structures Deep Dive

In backend development, choosing the wrong data structure will destroy your application's performance. For example, using `ArrayList.contains()` inside a loop over 100,000 leads will take **minutes**, while using a `HashSet` takes **5 milliseconds**.

---

## 1. The Java Collections Hierarchy

```
                    ┌──────────────┐
                    │   Iterable   │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Collection  │
                    └──────┬───────┘
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      ┌─────────────┐
   │    List     │  │     Set     │  │    Queue    │      │     Map     │ (Separate
   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘      └──────┬──────┘  Hierarchy)
          │                │                │                    │
   • ArrayList      • HashSet        • PriorityQueue      • HashMap
   • LinkedList     • TreeSet        • ArrayDeque         • TreeMap
                    • LinkedHashSet                       • ConcurrentHashMap
```

---

## 2. Lists: `ArrayList` vs. `LinkedList`

* **`ArrayList` (The Default Choice 99% of the Time)**:
  * Uses an internal dynamic array (`Object[]`).
  * **Lookup by Index (`get(i)`)**: `O(1)` instant access.
  * **Add to end**: Amortized `O(1)`. When full, it creates a new array 1.5x larger and copies elements (`Arrays.copyOf`).
  * **Add/Remove in middle**: `O(n)` because all subsequent elements must shift left/right.
* **`LinkedList` (Rarely Used in Modern Java)**:
  * Doubly linked nodes (`prev <- [Node] -> next`).
  * **Lookup by Index**: `O(n)` (must traverse from head or tail).
  * **Memory Overhead**: Heavy! Each node stores 3 object references (Data, Prev pointer, Next pointer). On 64-bit JVMs, it wastes 4x more RAM than an `ArrayList`!

---

## 3. The Internal Anatomy of `HashMap` (The #1 Interview Topic)

How does `map.put(key, value)` actually work under the hood?

```
Bucket Array (Default capacity = 16)
Index
[0] ──► null
[1] ──► Node(Key: "lead-1", Hash: 4912) ──► Node(Key: "lead-17", Hash: 8193) [Linked List]
[2] ──► null
...
[8] ──► TreeNode(Red-Black Balanced Tree if chain length >= 8)
```

### The Step-by-Step Execution of `put(key, value)`:
1. **Hash Calculation**: Calls `key.hashCode()`. Java applies a spread function (`h ^ (h >>> 16)`) to distribute bits evenly.
2. **Bucket Index Computation**: Calculates array index using bitwise AND:
   `index = (array_length - 1) & hash` (Much faster than modulo `%`).
3. **Collision Handling**:
   * If the bucket is empty: Insert a new `Node`.
   * If a collision occurs (two different keys map to the same bucket index):
     * Java traverses the bucket’s Linked List.
     * Checks `if (node.hash == hash && (node.key == key || node.key.equals(key)))`.
     * If true, overwrites value.
     * If false, appends to the tail.
4. **Treeification (Java 8+ Optimization)**:
   * If a single bucket's collision chain reaches **8 elements** (and table capacity >= 64), Java converts the linked list into a **Red-Black Self-Balancing Tree**.
   * Search time drops from `O(n)` worst-case down to `O(log n)`!
5. **Rehashing & Resizing**:
   * When element count exceeds `capacity * load_factor` (`16 * 0.75 = 12 elements`), the table size doubles to 32, and all keys are rehashed.

---

## 4. `ConcurrentHashMap` vs. `Collections.synchronizedMap`

* **`Collections.synchronizedMap()`**: Places a single giant lock over the **entire map**. If Thread A is reading, Thread B cannot write. It causes massive thread contention under load.
* **`ConcurrentHashMap` (Production Standard)**:
  * Uses **Bucket-Level Locking** and **CAS (Compare-And-Swap)** instructions.
  * Reads are completely lock-free (`volatile` reads).
  * Writes only lock the specific bucket being updated (`synchronized(node)` on the bucket head).
  * 16 threads can write to 16 different buckets simultaneously without blocking each other!

---

## 5. `ConcurrentModificationException` & Iteration Rules

**The Bug**: Modifying a collection while looping through it:
```java
List<String> leads = new ArrayList<>(List.of("Aarav", "Priya", "Rohan"));

for (String lead : leads) {
    if (lead.equals("Priya")) {
        leads.remove(lead); // CRASHES with ConcurrentModificationException!
    }
}
```
**Why?** `ArrayList` uses an internal `modCount` counter. The iterator detects that the list was modified outside of the iterator's knowledge (**Fail-Fast Iterator**).

### The Two Solutions:
1. **Use `Iterator.remove()`**:
   ```java
   Iterator<String> it = leads.iterator();
   while (it.hasNext()) {
       if (it.next().equals("Priya")) it.remove(); // SAFE!
   }
   ```
2. **Use Modern `removeIf()` (Java 8+)**:
   ```java
   leads.removeIf(lead -> lead.equals("Priya")); // Cleanest & Thread-Safe!
   ```

---

## 6. Collections Big-O Complexity Cheat Sheet

| Data Structure | Access by Index | Search by Value | Insert / Add | Delete |
| :--- | :---: | :---: | :---: | :---: |
| **`ArrayList`** | **`O(1)`** | `O(n)` | `O(1)` (amortized) | `O(n)` |
| **`LinkedList`**| `O(n)` | `O(n)` | **`O(1)`** (at head/tail) | **`O(1)`** |
| **`HashSet`**   | N/A | **`O(1)`** | **`O(1)`** | **`O(1)`** |
| **`TreeSet`**   | N/A | `O(log n)` | `O(log n)` | `O(log n)` |
| **`HashMap`**   | N/A (Key) | **`O(1)`** (by key) | **`O(1)`** | **`O(1)`** |
| **`TreeMap`**   | N/A (Key) | `O(log n)` | `O(log n)` | `O(log n)` |
