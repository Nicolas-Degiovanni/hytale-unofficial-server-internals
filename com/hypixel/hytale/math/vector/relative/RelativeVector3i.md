---
description: Architectural reference for RelativeVector3i
---

# RelativeVector3i

**Package:** com.hypixel.hytale.math.vector.relative
**Type:** Value Object

## Definition
```java
// Signature
public class RelativeVector3i {
```

## Architecture & Concepts

The RelativeVector3i class is a specialized data structure designed to represent a 3D integer coordinate that can be interpreted in one of two ways: as an *absolute* position or as a *relative* offset from a separate, context-supplied origin point.

Its primary role is within the data layer of the engine, particularly for serialization and configuration. The presence of a static **CODEC** field indicates that this object is a fundamental building block for defining game data, such as entity attachment points, particle effect origins, or procedural generation rules. By encoding a position as "relative" or "absolute", game designers can create flexible templates that adapt to their runtime context. For example, defining a weapon's muzzle flash position relative to the weapon's model origin.

This class encapsulates the data and the logic to transform it into a concrete, world-space coordinate through its **resolve** method. It acts as a descriptor, not a live-world object.

## Lifecycle & Ownership

-   **Creation:** RelativeVector3i instances are not managed as singletons or services. They are typically created in one of two ways:
    1.  **Deserialization:** The most common path. The engine's **Codec** system instantiates the object from a data source (e.g., a JSON or binary asset file) using the public static **CODEC** field.
    2.  **Direct Instantiation:** Programmatic creation via `new RelativeVector3i(vector, isRelative)` for runtime-defined logic.
-   **Scope:** These objects are transient and short-lived. They exist to hold configuration data and are typically discarded after their information is used to calculate a final **Vector3i** coordinate. They are not intended to be held as long-term state.
-   **Destruction:** Ownership is transient. Once all references are dropped, the object is reclaimed by the Java Garbage Collector. No manual destruction or cleanup is necessary.

## Internal State & Concurrency

-   **State:** The object's state is defined by its internal **vector** and **relative** fields. While the fields are not declared final, the object is intended to be treated as immutable after its initial creation or deserialization. The **CODEC** implementation relies on a protected no-arg constructor and subsequent field population, which necessitates mutable fields.

-   **Thread Safety:** This class is **not thread-safe**. It is a plain data holder with no internal synchronization.
    -   **WARNING:** Do not share and modify an instance of RelativeVector3i across multiple threads without external locking. The intended pattern is to deserialize or create it, read from it within a single thread, and then discard it.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(1) | Static codec for serialization and deserialization. The primary entry point for data-driven creation. |
| resolve(Vector3i vector) | Vector3i | O(1) | Calculates a final coordinate based on a provided origin vector and the object's internal state. |
| getVector() | Vector3i | O(1) | Returns the internal vector, which represents either an absolute position or a relative offset. |
| isRelative() | boolean | O(1) | Returns true if the internal vector should be treated as a relative offset. |

## Integration Patterns

### Standard Usage

The primary use case is to resolve a defined position against a runtime origin. This is common in entity logic, where an attachment point is resolved against the entity's current position.

```java
// Assume 'entityPosition' is the current location of an entity
Vector3i entityPosition = new Vector3i(100, 50, 100);

// 'attachmentPoint' is loaded from an entity definition file
// This represents a point 5 units above the origin
RelativeVector3i attachmentPoint = new RelativeVector3i(new Vector3i(0, 5, 0), true);

// The 'resolve' method should compute the final world coordinate.
// Expected behavior: worldPosition becomes (100, 55, 100)
Vector3i worldPosition = attachmentPoint.resolve(entityPosition);
```

### Anti-Patterns (Do NOT do this)

-   **State Mutation:** Do not modify the internal state of a RelativeVector3i object after it has been created. It should be treated as an immutable value object.
-   **Misinterpretation of Resolve:** The **resolve** method's behavior is highly counter-intuitive and does not align with the class's conceptual purpose.

    -   **WARNING:** The current implementation of **resolve(Vector3i vector)** completely ignores the object's internal **this.vector** field. Its logic is solely dependent on the **isRelative** flag and the *input* vector.
        -   If **isRelative()** is true, it returns the input vector doubled (`vector.clone().add(vector)`).
        -   If **isRelative()** is false, it returns a clone of the input vector.
    -   This behavior is almost certainly a bug or placeholder code. Relying on it is extremely dangerous and will likely lead to incorrect positional calculations. Any usage must be paired with a thorough code review to validate its intent.

## Data Pipeline

RelativeVector3i serves as a data container within a larger configuration and instantiation pipeline. It does not actively process data streams but is rather the product of one.

> Flow:
> Data Source (JSON/Binary Asset) -> Engine's Codec System -> **RelativeVector3i Instance** -> Game Logic (e.g., Entity Spawner) -> `resolve()` call with origin -> Final World Coordinate (Vector3i)<ctrl63>

