---
description: Architectural reference for RelativeVector3d
---

# RelativeVector3d

**Package:** com.hypixel.hytale.math.vector.relative
**Type:** Value Object / Data Transfer Object (DTO)

## Definition
```java
// Signature
public class RelativeVector3d {
```

## Architecture & Concepts
The RelativeVector3d class is a fundamental data structure within the engine's coordinate and positioning system. It encapsulates a 3D vector that can represent either an **absolute** world position or a **relative** offset from a given reference point.

This distinction is controlled by the internal boolean flag, *relative*, which dictates the behavior of the core `resolve` method. This dual-purpose design allows game data, such as entity attachment points or particle effect origins, to be defined in a flexible, context-agnostic manner.

The class is designed for serialization and is a key component of the data-driven architecture. The static **CODEC** field provides the necessary metadata for the engine's serialization framework to encode and decode instances of RelativeVector3d from configuration files or network streams. It acts as a passive data container whose primary purpose is to be instantiated, transported, and then resolved into an absolute world-space coordinate by game logic.

## Lifecycle & Ownership
- **Creation:** Instances are ephemeral and created on-demand. The primary creation path is through the engine's serialization framework, which uses the public **CODEC** field and the protected no-arg constructor. Instances are also created directly in game code to define positional logic.
- **Scope:** The lifetime of a RelativeVector3d instance is short and typically scoped to its containing object, such as an entity component or an animation definition. It is not a globally managed object.
- **Destruction:** Instances are managed by the Java garbage collector and are destroyed once all references are released. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** The object's state consists of a Vector3d and a boolean. While the class provides no public setters, the fields are not final, and the serialization mechanism mutates the object during creation. After hydration, it should be treated as an **effectively immutable** value object.
- **Thread Safety:** The class is **conditionally thread-safe**. Once an instance is fully constructed, it can be safely read and shared across multiple threads. The `resolve` method is a pure function that returns a new object, making it inherently safe for concurrent invocation.

**WARNING:** Modifying the internal state of a shared RelativeVector3d instance after its initial construction is not a supported pattern and will lead to race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| resolve(Vector3d vector) | Vector3d | O(1) | Calculates a final position based on the input vector. If relative, it adds the input vector to itself. If not, it returns a clone of the input. |
| getVector() | Vector3d | O(1) | Returns the internal vector, which represents either an absolute position or a relative offset. |
| isRelative() | boolean | O(1) | Returns true if the internal vector should be treated as a relative offset. |

**WARNING:** The current implementation of `resolve` when `isRelative` is true is `vector.clone().add(vector)`. This doubles the input vector, which may not be the intended behavior. The expected behavior for resolving a relative offset is typically `vector.clone().add(this.vector)`. Developers must be aware of this implementation detail.

## Integration Patterns

### Standard Usage
The primary use case is to resolve a relative position into an absolute world-space coordinate based on a reference point, such as an entity's location.

```java
// Example: Calculate the position for a particle effect on a player's head.
Vector3d playerPosition = player.getPosition(); // Assume this is (100, 50, 200)

// This object, loaded from a config file, defines a 1.8 unit offset on the Y axis.
RelativeVector3d headOffset = new RelativeVector3d(new Vector3d(0, 1.8, 0), true);

// The 'resolve' method is intended to calculate the final world position.
// NOTE: Due to the current implementation, this example uses the presumed intended logic.
Vector3d finalEffectPosition = playerPosition.clone().add(headOffset.getVector());
// finalEffectPosition is now (100, 51.8, 200)
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not retrieve the internal Vector3d via `getVector` and modify it. This breaks the assumption of immutability and can cause unpredictable side effects in other systems that hold a reference to the same RelativeVector3d instance.
- **Discarding Return Value:** The `resolve` method does not modify the instance it is called on, nor the vector passed to it. It returns a **new** Vector3d. Failing to capture this return value makes the call useless.

## Data Pipeline
RelativeVector3d is not a processing component itself, but rather a data payload that moves through various systems.

> Flow:
> Serialized Data (e.g., JSON/Binary) -> Engine Codec Framework -> **RelativeVector3d Instance** -> Game Logic (e.g., Entity System) -> `resolve` call -> Final World Position (Vector3d) -> Physics/Rendering Engine

