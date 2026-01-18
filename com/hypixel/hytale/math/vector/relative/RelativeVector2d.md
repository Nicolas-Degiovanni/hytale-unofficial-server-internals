---
description: Architectural reference for RelativeVector2d
---

# RelativeVector2d

**Package:** com.hypixel.hytale.math.vector.relative
**Type:** Value Object

## Definition
```java
// Signature
public class RelativeVector2d {
```

## Architecture & Concepts
The RelativeVector2d class is a fundamental data structure used throughout the engine to represent a 2D coordinate that can be interpreted in one of two ways: absolute or relative. Its primary role is to serve as a serializable data container, bridging configuration files or network data with in-game logic.

The presence of a static **CODEC** field is the most critical architectural feature. This indicates that RelativeVector2d is designed to be seamlessly integrated with Hytale's serialization framework. It is not a service or manager, but rather a payload object used to define spatial properties of entities, UI elements, or particle effects in a data-driven manner.

The core concept is to defer the calculation of a final position. A system can define a point as either a fixed coordinate (e.g., 100, 100) or as an offset from a dynamic base point (e.g., +10, -5 from the player's current position). This class encapsulates that choice.

## Lifecycle & Ownership
-   **Creation:** Instances are primarily created by the Hytale **Codec** system during deserialization of game data (e.g., from JSON or binary asset files). They can also be instantiated directly in code when a relative position needs to be programmatically defined.
-   **Scope:** Transient and short-lived. A RelativeVector2d instance typically exists as a field within a larger configuration object or component state. It does not manage its own lifecycle.
-   **Destruction:** The object is managed by the Java garbage collector. It is cleaned up when its containing object is no longer referenced. No manual destruction or cleanup is necessary.

## Internal State & Concurrency
-   **State:** Mutable. The internal fields are not final to support the builder pattern employed by the **BuilderCodec** for efficient deserialization. However, after initial construction, an instance should be treated as an immutable value object. Modifying its state post-creation is a severe anti-pattern and can lead to unpredictable behavior.
-   **Thread Safety:** This class is **not thread-safe**. It is a plain data container with no internal synchronization. It must not be shared and modified across multiple threads. Its intended use is within a single-threaded context, such as the main game update loop, or as a temporary object during a data loading process.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVector() | Vector2d | O(1) | Retrieves the internal vector component. |
| isRelative() | boolean | O(1) | Returns true if the vector should be treated as a relative offset. |
| resolve(Vector2d vector) | Vector2d | O(1) | Calculates a final, absolute vector based on a provided base vector. |

**WARNING:** The current implementation of the resolve method does **not** use the internal vector state of this object. If `isRelative` is true, it returns a clone of the input vector added to itself (effectively doubling it). If false, it returns a clone of the input vector. This behavior is counter-intuitive and likely incorrect. Systems using this method must be aware of this implementation detail.

## Integration Patterns

### Standard Usage
The primary use case is to define a position in data and resolve it against a context-specific base position in code.

```java
// Assume 'parentEntityPosition' is a dynamic value from the game world
Vector2d parentEntityPosition = new Vector2d(500.0, 200.0);

// This RelativeVector2d would typically be loaded from a config file.
// Here, we define a relative offset of (10, 25).
RelativeVector2d attachmentPoint = new RelativeVector2d(new Vector2d(10.0, 25.0), true);

// To get the final world position, you must perform the resolution manually
// due to the flawed 'resolve' method.
Vector2d finalPosition;
if (attachmentPoint.isRelative()) {
    finalPosition = parentEntityPosition.clone().add(attachmentPoint.getVector());
} else {
    finalPosition = attachmentPoint.getVector().clone();
}
```

### Anti-Patterns (Do NOT do this)
-   **Relying on the resolve method:** Do not call `attachmentPoint.resolve(parentEntityPosition)`. As documented, it ignores the internal state of the `attachmentPoint` object and will produce incorrect results for calculating offsets.
-   **Post-Construction Mutation:** Do not modify the state of a RelativeVector2d after it has been created and passed to other systems. Treat it as immutable.
-   **Concurrent Access:** Do not share a single instance across multiple threads. If a thread needs this data, pass it a copy.

## Data Pipeline
RelativeVector2d acts as a data payload that is hydrated from a persistent format and then consumed by game logic.

> Flow:
> JSON/Binary Asset -> Hytale Codec System -> **RelativeVector2d Instance** -> Game Logic (e.g., PositionSystem) -> Manual Resolution -> Final Vector2d

