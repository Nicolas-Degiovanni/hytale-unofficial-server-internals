---
description: Architectural reference for BoolIntPair
---

# BoolIntPair

**Package:** com.hypixel.hytale.common.tuple
**Type:** Value Object / Data Structure

## Definition
```java
// Signature
public class BoolIntPair implements Comparable<BoolIntPair> {
```

## Architecture & Concepts
The BoolIntPair is a foundational, low-level data structure designed to serve as a specialized, type-safe tuple. Its primary architectural role is to provide an **immutable container** for a primitive boolean and a primitive integer, avoiding the performance overhead and potential nullability issues associated with the generic `Pair<Boolean, Integer>`.

By implementing standard Java contracts such as `Comparable`, `equals`, and `hashCode`, this class is engineered for reliable use within core Java Collections. It is frequently used as a key in hash-based maps or as an element in sorted sets where a compound key composed of a boolean flag and an integer value is required. Its existence promotes code clarity by explicitly defining the contained types, unlike more generic tuple implementations.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand whenever a boolean and an integer need to be logically grouped. The static factory method `of` is the conventional and preferred mechanism for instantiation.
- **Scope:** Transient. A BoolIntPair object is typically short-lived, existing within the scope of a method or as a temporary data carrier. Its lifetime is managed entirely by the Java Garbage Collector.
- **Destruction:** The object is eligible for garbage collection as soon as no strong references to it exist. No manual cleanup or resource management is necessary.

## Internal State & Concurrency
- **State:** **Immutable**. Both the `left` (boolean) and `right` (int) fields are declared `final` and are initialized only once within the constructor. Once an instance is created, its state can never be modified. This is a critical design guarantee.
- **Thread Safety:** **Inherently thread-safe**. Due to its strict immutability, a BoolIntPair instance can be safely shared and read by multiple threads concurrently without any external synchronization or locking mechanisms. This makes it an ideal candidate for use in multi-threaded contexts, such as caching or concurrent data processing.

## API Surface
The public contract is minimal and focused on data access and comparison. The dual-named accessors (e.g., `getKey` and `getLeft`) provide semantic flexibility for different use cases, such as treating it as a map entry or a simple tuple.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(left, right) | static BoolIntPair | O(1) | **[Factory]** Constructs a new instance. This is the preferred method of creation. |
| getKey() / getLeft() | boolean | O(1) | Returns the left-hand boolean value. |
| getValue() / getRight() | int | O(1) | Returns the right-hand integer value. |
| compareTo(other) | int | O(1) | Compares this pair to another. The boolean value is the primary sort key. |
| equals(o) | boolean | O(1) | Performs a value-based equality check. |
| hashCode() | int | O(1) | Computes a hash code based on both internal values. |

## Integration Patterns

### Standard Usage
The most common use case is to create a temporary grouping of a flag and a value, often for use in collections or as a return type from a private helper method.

```java
// Example: Storing task completion status and duration in a set
Set<BoolIntPair> taskResults = new HashSet<>();

// The 'of' factory method is the canonical way to create an instance
taskResults.add(BoolIntPair.of(true, 120)); // Succeeded in 120ms
taskResults.add(BoolIntPair.of(false, 5000)); // Failed after 5000ms

// The object is immediately usable in logic
for (BoolIntPair result : taskResults) {
    if (result.getKey() == false) {
        System.err.println("A task failed with duration: " + result.getValue());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Public API Exposure:** Avoid using BoolIntPair in public-facing API signatures. A dedicated, named class (e.g., `TaskResult`) provides significantly more context and type safety for consumers. Reserve BoolIntPair for internal implementation details.
- **Mutable Wrappers:** Do not wrap a BoolIntPair in another object to simulate mutability. The immutability guarantee is a core feature for ensuring thread safety and predictable behavior. If mutable state is required, design a different class for that purpose.
- **Complex State Representation:** Do not use this class to represent complex state that involves more than two tightly-coupled primitive values. Doing so leads to unreadable and unmaintainable code.

## Data Pipeline
As a simple value object, BoolIntPair does not process data itself. Instead, it acts as a **data payload** that is passed between different systems or stored in collections. It is a container, not a transformer.

> **Flow:**
>
> Game Event Logic -> **BoolIntPair (Payload Creation)** -> Data Collection (e.g., `HashMap<EntityID, BoolIntPair>`) -> State Aggregator -> Final Processing

