---
description: Architectural reference for RelativeVector2i
---

# RelativeVector2i

**Package:** com.hypixel.hytale.math.vector.relative
**Type:** Value Object / Transient

## Definition
```java
// Signature
public class RelativeVector2i {
```

## Architecture & Concepts
The RelativeVector2i class is a fundamental data structure representing a 2D integer vector that can be interpreted as either an absolute coordinate or a relative offset. Its primary role is to decouple a spatial definition from its final application in the game world, providing flexibility in configuration and procedural generation.

This class is not a service or a manager; it is a plain data-carrier object, often referred to as a Value Object. Its design is heavily influenced by the engine's serialization framework, as evidenced by the static CODEC field. This allows instances of RelativeVector2i to be seamlessly loaded from data files (e.g., JSON definitions for entities, UI layouts, or particle effects) and transmitted over the network.

The core architectural concept is the boolean flag *relative*.
- When **false**, the internal Vector2i is treated as an absolute position.
- When **true**, the internal Vector2i is treated as an offset to be applied to a base position.

This pattern is used extensively to define, for example, the spawn point of a particle effect relative to its emitting entity, or the position of a UI element relative to its parent container.

## Lifecycle & Ownership
- **Creation:** Instances are created in two primary scenarios:
    1.  **Programmatically:** Via direct instantiation with `new RelativeVector2i(vector, isRelative)` during gameplay logic.
    2.  **Declaratively:** By the engine's serialization system (using the defined CODEC) when parsing configuration files or network packets. The protected no-arg constructor exists exclusively to support this deserialization pathway.

- **Scope:** Transient. A RelativeVector2i object has no managed lifecycle and exists only as long as it is referenced. It is typically held as a field within a larger configuration object or as a local variable within a method.

- **Destruction:** The object is managed by the Java garbage collector and is destroyed when it is no longer reachable. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The internal state consists of a Vector2i and a boolean. While the fields are not declared as final, the absence of public setters makes the class **effectively immutable**. Consumers should treat instances as immutable values.

    **Warning:** The internal Vector2i object is itself mutable. Modifying the object returned by getVector is a severe anti-pattern that violates the immutability contract and can lead to unpredictable, non-local side effects.

- **Thread Safety:** The class is inherently thread-safe for read operations due to its effective immutability. The resolve method is a pure function, producing a new Vector2i instance without modifying internal state or causing side effects. Instances can be safely shared and read across multiple threads without synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVector() | Vector2i | O(1) | Returns the internal vector component. |
| isRelative() | boolean | O(1) | Returns true if the vector should be treated as a relative offset. |
| resolve(Vector2i vector) | Vector2i | O(1) | Calculates a final vector based on a provided base vector. **Warning:** See detailed notes below. |

### The resolve Method
The `resolve` method is the primary behavior of this class, but its implementation in the provided source is highly counter-intuitive and likely contains a logical error.

**Actual Behavior:**
- If `isRelative()` is true, the method returns the **input** vector added to itself (`vector * 2`).
- If `isRelative()` is false, the method returns a clone of the **input** vector.

**Warning:** In its current form, the `resolve` method **completely ignores the internal state** of the RelativeVector2i instance (the vector stored within it). This behavior is inconsistent with the class's name and purpose. Developers should exercise extreme caution and verify its behavior, as the expected logic would be to use the internal vector, not the input vector, for calculations.

## Integration Patterns

### Standard Usage
The intended pattern is to deserialize this object from a configuration source and use it to calculate a final position from a dynamic base position (like an entity's location).

```java
// Assume 'entityConfig' was loaded from a JSON file by the engine
RelativeVector2i attachmentPoint = entityConfig.getAttachmentPoint();
Vector2i entityPosition = entity.getWorldPosition();

// The 'resolve' method is used to get the final world coordinate
// NOTE: Be aware of the flawed logic described in the API Surface section.
Vector2i finalAttachmentPosition = attachmentPoint.resolve(entityPosition);

// Use the final, absolute position for game logic
spawnEffectAt(finalAttachmentPosition);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not retrieve the internal vector and modify it. This breaks the immutability contract and can cause difficult-to-trace bugs.
    ```java
    // DO NOT DO THIS
    RelativeVector2i pos = config.getPosition();
    pos.getVector().setX(100); // This modifies the shared config state!
    ```

- **Assuming Resolve Logic:** Do not assume the `resolve` method functions as its name implies. As documented, it ignores the object's internal vector. Relying on it to apply a relative offset from `this.vector` will fail. Always validate its behavior in your context.

## Data Pipeline
The RelativeVector2i class is a common component in the engine's data serialization and configuration pipeline.

> Flow:
> Configuration File (JSON) -> Engine Codec System -> **RelativeVector2i** instance -> Game Logic (e.g., Entity Factory) -> `resolve()` called with base position -> Final World Coordinate

