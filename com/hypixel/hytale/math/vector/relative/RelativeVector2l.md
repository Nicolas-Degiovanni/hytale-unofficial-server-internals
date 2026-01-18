---
description: Architectural reference for RelativeVector2l
---

# RelativeVector2l

**Package:** com.hypixel.hytale.math.vector.relative
**Type:** Transient

## Definition
```java
// Signature
public class RelativeVector2l {
```

## Architecture & Concepts
RelativeVector2l is a specialized data structure that represents a two-dimensional vector whose value can be interpreted as either an absolute coordinate or a relative offset. This distinction is fundamental to Hytale's data-driven content pipeline, allowing designers and systems to define positions without hard-coding them to a specific frame of reference.

The primary architectural features are:

1.  **Data-Driven Positioning:** The class includes a static **CODEC** field. This exposes it to the engine's serialization system, enabling its use directly within configuration files (e.g., JSON). Designers can specify a particle emitter's position as `relative: true` to an entity's bone, or a UI element's position as `relative: false` for a fixed screen coordinate.

2.  **Semantic Encapsulation:** It encapsulates not just a Vector2l, but the *intent* behind that vector. The boolean flag *relative* provides the context needed to correctly interpret the vector's value at runtime.

3.  **Resolution Logic:** The core functionality is the **resolve** method, which transforms the stored vector into a final, absolute world coordinate by taking a base vector as input. This decouples the definition of a position from its application.

## Lifecycle & Ownership
-   **Creation:** Instances are overwhelmingly created by the engine's **Codec** system during the deserialization of asset files. Programmatic creation via `new RelativeVector2l(...)` is also supported for procedural content generation. The protected no-argument constructor exists exclusively for use by the serialization framework.

-   **Scope:** This is a value object. Its lifetime is strictly bound to its owning component (e.g., an entity template, a UI layout definition, a particle effect asset). Instances are expected to be short-lived and frequently instantiated.

-   **Destruction:** Managed entirely by the Java Garbage Collector. When the parent object that holds a reference to a RelativeVector2l is garbage collected, the instance is destroyed. No manual memory management is required.

## Internal State & Concurrency
-   **State:** The internal state consists of a Vector2l and a boolean flag. While the fields are not declared final, instances of this class should be treated as **effectively immutable** after they are initialized by the Codec system or a constructor. Modifying the internal state after creation is an anti-pattern that can lead to unpredictable behavior.

-   **Thread Safety:** This class is **not thread-safe**. It contains no synchronization mechanisms. It is designed for use within a single, well-defined thread context, such as the main game thread or a content loading thread. Concurrent access from multiple threads without external locking will result in data races and undefined behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(1) | **Critical:** Static codec for serialization and deserialization from data files. |
| resolve(Vector2l) | Vector2l | O(1) | Calculates the final absolute position based on a provided base vector. |
| getVector() | Vector2l | O(1) | Returns the internal vector component. |
| isRelative() | boolean | O(1) | Returns true if the vector should be treated as a relative offset. |

**WARNING:** The `resolve` method's implementation ignores the internal `vector` field entirely. If `isRelative` is true, it returns the input vector doubled. If false, it returns a clone of the input vector. This behavior is likely incorrect and may be subject to change.

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve a RelativeVector2l instance from a configuration-driven object and use it to calculate a final world-space position based on a dynamic context, such as an entity's current location.

```java
// Assume 'effectDefinition' is loaded from a config file
RelativeVector2l spawnOffset = effectDefinition.getSpawnOffset();
Vector2l entityPosition = entity.getPosition();

// Resolve the offset relative to the entity's position
Vector2l finalSpawnPosition = spawnOffset.resolve(entityPosition);

world.spawnParticle(finalSpawnPosition);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Do not modify the internal state of a RelativeVector2l after it has been created. This violates its contract as a value object and can cause difficult-to-trace bugs in systems that share the reference.
-   **Incorrect Resolution Context:** Passing an incorrect base vector to the `resolve` method will produce a logically incorrect, but non-crashing, result. The context for resolution is critical.
-   **Cross-Thread Sharing:** Never share an instance across threads without explicit and robust external synchronization.

## Data Pipeline
RelativeVector2l acts as a data transformation stage, converting declarative configuration data into concrete spatial data at runtime.

> Flow:
> Asset File (JSON) -> Engine Codec System -> **RelativeVector2l Instance** -> Game Logic calls `resolve` with context -> Final World Coordinate (Vector2l)

