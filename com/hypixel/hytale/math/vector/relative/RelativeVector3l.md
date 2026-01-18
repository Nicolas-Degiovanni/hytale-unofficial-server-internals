---
description: Architectural reference for RelativeVector3l
---

# RelativeVector3l

**Package:** com.hypixel.hytale.math.vector.relative
**Type:** Transient

## Definition
```java
// Signature
public class RelativeVector3l {
```

## Architecture & Concepts
The RelativeVector3l class is a fundamental data structure that represents a 3D coordinate with a long integer precision, which can be interpreted as either an absolute position or a relative offset. Its primary architectural role is to provide a unified type for defining locations within the game world, particularly in data-driven contexts like entity definitions, particle effect configurations, or procedural generation rules.

This class decouples a coordinate value from its frame of reference. By encapsulating a Vector3l and a boolean *relative* flag, the system can serialize and deserialize positions without needing prior context. The consumer of this object is responsible for providing the base context (e.g., an entity's current position) to resolve the vector into an absolute world coordinate.

The inclusion of a static **CODEC** field is a critical design feature, indicating that this class is a first-class citizen in the engine's serialization and data-binding framework. It is designed to be read from and written to network packets, configuration files, and saved game state.

## Lifecycle & Ownership
-   **Creation:** Instances are created on-demand. The primary mechanism for instantiation is via the engine's serialization system, which uses the public static CODEC to construct objects from a data source. Manual instantiation via `new RelativeVector3l(...)` is common in game logic for defining dynamic offsets.
-   **Scope:** Short-lived and transient. A RelativeVector3l instance is a value object; it does not manage any resources and is intended to exist only as long as it is referenced by a component that needs it. It is not a managed service.
-   **Destruction:** The object is managed by the Java garbage collector and is destroyed when it is no longer reachable. No manual cleanup or destruction methods are required.

## Internal State & Concurrency
-   **State:** The internal state consists of a Vector3l and a boolean flag. While the class itself does not expose setters, the underlying Vector3l object it holds is mutable. Best practice dictates that instances of RelativeVector3l should be treated as **immutable** after creation to prevent side effects.
-   **Thread Safety:** This class is **not thread-safe**. It is a plain data container with no internal locking or synchronization. It is intended for use within a single thread, such as the main game loop or a dedicated world-processing thread. Sharing instances across threads without external synchronization will lead to undefined behavior.

## API Surface
The public API is minimal, focusing on data access and the core resolution logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| resolve(Vector3l vector) | Vector3l | O(1) | **Core Operation.** Resolves the vector into an absolute world coordinate. If relative, it adds its internal vector to the supplied base vector. If absolute, it returns a clone of its own internal vector. |
| getVector() | Vector3l | O(1) | Returns the internal Vector3l component. |
| isRelative() | boolean | O(1) | Returns true if the internal vector represents a relative offset. |

## Integration Patterns

### Standard Usage
The most common use case is to define a target position or offset in data and then resolve it against a dynamic base position in game logic.

```java
// In a configuration file, this might define a particle effect's origin
// as 10 blocks above the source entity.
RelativeVector3l effectOffset = new RelativeVector3l(new Vector3l(0, 10, 0), true);

// In game logic, get the source entity's absolute position
Vector3l entityPosition = entity.getWorldPosition();

// Resolve the offset to get the absolute world coordinate for the effect
Vector3l absoluteEffectPosition = effectOffset.resolve(entityPosition);

// Spawn the particle effect at the calculated absolute position
ParticleEngine.spawn(EFFECT_ID, absoluteEffectPosition);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Do not retrieve the internal vector and modify it. This breaks the assumption of immutability and can cause unpredictable side effects in other systems that hold a reference to the same RelativeVector3l instance.
    ```java
    // DO NOT DO THIS
    RelativeVector3l vec = new RelativeVector3l(new Vector3l(5, 5, 5), true);
    Vector3l internal = vec.getVector();
    internal.setX(100); // This mutates the state of 'vec' unexpectedly
    ```
-   **Incorrect Resolution Context:** The `resolve` method expects an absolute world coordinate as its argument. Passing another relative vector or an uninitialized vector will produce incorrect results.

## Data Pipeline
RelativeVector3l is a passive data container, but it is a key component in the engine's data processing pipelines, especially for configuration and entity spawning.

> Flow:
> JSON/Binary Config File -> Engine Deserializer -> **BuilderCodec** -> **RelativeVector3l** instance -> Entity Spawner Logic -> resolve() -> Absolute World Coordinate -> Entity Transform Component

