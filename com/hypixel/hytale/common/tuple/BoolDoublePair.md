---
description: Architectural reference for BoolDoublePair
---

# BoolDoublePair

**Package:** com.hypixel.hytale.common.tuple
**Type:** Transient

## Definition
```java
// Signature
public class BoolDoublePair implements Comparable<BoolDoublePair> {
```

## Architecture & Concepts
The BoolDoublePair is a foundational, immutable data structure designed to encapsulate a primitive boolean and a primitive double into a single, cohesive object. It is a specialized Tuple, optimized for this specific type combination to avoid the overhead and type-casting associated with more generic tuple implementations.

Its core architectural principle is **immutability**. Once an instance is created, its internal state—the boolean and double values—cannot be changed. This design choice makes BoolDoublePair instances inherently thread-safe and allows them to be used as stable keys in collections or cached without risk of modification.

The class implements the Comparable interface, which enables collections of BoolDoublePair objects to be sorted. The comparison logic is deterministic: it first compares the boolean values (false is less than true), and only if they are equal does it proceed to compare the double values. This establishes a total ordering for all instances.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by any system requiring a compound return value or a simple key. The preferred construction mechanism is the static factory method `of(boolean, double)`, which improves readability over the direct constructor call.
- **Scope:** The lifetime of a BoolDoublePair is typically ephemeral. It is intended to be used as a method-local variable, a return type, or a transient data carrier. It is not designed for long-term persistence or serialization.
- **Destruction:** Management is handled entirely by the Java Garbage Collector. When an instance is no longer referenced, it becomes eligible for garbage collection. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** **Immutable**. The internal `left` (boolean) and `right` (double) fields are declared `final` and are assigned only once within the constructor. No public methods exist to modify the state after instantiation.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, a BoolDoublePair instance can be safely shared and read by multiple threads concurrently without any external synchronization or locking mechanisms. This is a critical guarantee for high-performance, multi-threaded systems.

## API Surface
The public API is minimal and focused on construction and data access. The aliases `getKey`/`getValue` provide semantic compatibility with map-like structures.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(boolean left, double right) | static BoolDoublePair | O(1) | Constructs a new instance. This is the preferred factory method. |
| getLeft() / getKey() | boolean | O(1) | Returns the encapsulated boolean value. |
| getRight() / getValue() | double | O(1) | Returns the encapsulated double value. |
| compareTo(BoolDoublePair other) | int | O(1) | Compares this pair to another, first by boolean then by double. |
| equals(Object o) | boolean | O(1) | Performs a value-based equality check. |
| hashCode() | int | O(1) | Computes a hash code based on the contained values. |

## Integration Patterns

### Standard Usage
BoolDoublePair is most effective as a return type for methods that need to convey both a status (the boolean) and an associated metric (the double).

```java
// A method in a physics or AI system might return a pair
// indicating if a target is visible and its distance.
public BoolDoublePair checkLineOfSight(Entity target) {
    boolean isVisible = performRaycast(target);
    if (isVisible) {
        double distance = calculateDistance(target);
        return BoolDoublePair.of(true, distance);
    }
    return BoolDoublePair.of(false, Double.MAX_VALUE);
}

// Consumer code
BoolDoublePair sightResult = checkLineOfSight(enemy);
if (sightResult.getKey()) {
    System.out.println("Target is visible at distance: " + sightResult.getValue());
}
```

### Anti-Patterns (Do NOT do this)
- **Wrapping for Mutability:** Do not create a mutable wrapper class around a BoolDoublePair to change its values. If mutable state is required, use a dedicated, mutable class. The immutability of this pair is a deliberate and critical design feature.
- **Overuse in Complex APIs:** For functions returning more than two conceptually distinct values, a BoolDoublePair can become unreadable. In such cases, define a dedicated, named data-transfer object (DTO) to improve code clarity and maintainability.

## Data Pipeline
As a simple data carrier, BoolDoublePair does not process data itself but is often the *payload* that flows between systems. It acts as a container for the output of one stage to be used as the input for the next.

> **Flow Example: AI Targeting Logic**
>
> AI Sensor System -> **BoolDoublePair (isHostile, threatScore)** -> AI Decision Engine -> Action Scheduler

