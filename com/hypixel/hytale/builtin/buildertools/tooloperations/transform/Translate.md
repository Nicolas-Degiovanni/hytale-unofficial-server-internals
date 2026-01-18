---
description: Architectural reference for Translate
---

# Translate

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations.transform
**Type:** Value Object / Strategy

## Definition
```java
// Signature
public class Translate implements Transform {
```

## Architecture & Concepts
The Translate class is a concrete implementation of the **Strategy Pattern**, defined by the Transform interface. It encapsulates a single, atomic 3D translation operation. Within the Hytale engine, this class serves as a fundamental component of the builder tools framework, representing the mathematical concept of moving a point or a set of points by a fixed offset.

Its primary architectural role is to provide a stateless, reusable operation that can be composed with other Transform objects (like Rotate or Scale) to create complex procedural generation or world editing effects. The design decision to have the apply method mutate the input Vector3i is a critical performance optimization, avoiding the significant overhead of object allocation within tight loops that process large volumes of voxel or entity data.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively via the static factory methods, `Translate.of(...)`. This is typically invoked by a higher-level tool operation manager when parsing user commands or executing a build script. The factory includes a flyweight optimization, returning a shared `Transform.NONE` instance for zero-vector translations to reduce object churn.
- **Scope:** Transient and short-lived. A Translate object's lifetime is typically bound to the execution of a single build tool operation. It does not persist between operations or game sessions.
- **Destruction:** Managed by the Java Garbage Collector. Once the composite operation that created it is complete, the Translate instance is no longer referenced and becomes eligible for collection.

## Internal State & Concurrency
- **State:** Immutable. The internal integer coordinates are declared as final and are set once in the private constructor. This immutability is a core design feature, ensuring that a defined translation operation cannot be accidentally modified after creation.
- **Thread Safety:** Inherently thread-safe. Because its internal state cannot change, a single Translate instance can be safely shared and used by multiple threads simultaneously without locks or synchronization.
    - **Warning:** The object passed to the `apply` method, `Vector3i`, is **mutated** by the operation. The thread safety of the *overall operation* depends on the external synchronization of the `Vector3i` instance, not the Translate object itself.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(Vector3i vector3i) | void | O(1) | Mutates the provided vector by adding the translation offset. This is an in-place modification. |
| of(Vector3i vector) | Transform | O(1) | Primary factory method. Creates a Translate instance from a given vector. |
| of(int x, int y, int z) | Transform | O(1) | Primary factory method. Creates a Translate instance from raw coordinates. Returns a shared no-op instance if all inputs are zero. |

## Integration Patterns

### Standard Usage
Translate is designed to be used as part of a larger transformation pipeline. A developer should obtain an instance from the factory and apply it to a target vector.

```java
// A vector representing a block's position
Vector3i blockPosition = new Vector3i(10, 50, 20);

// Create a transform to move the block 5 units up the Y axis
Transform moveUp = Translate.of(0, 5, 0);

// Apply the transformation. The original blockPosition object is modified.
moveUp.apply(blockPosition);

// blockPosition is now (10, 55, 20)
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Attempting to instantiate with `new Translate(...)` will result in a compile-time error. Always use the static `Translate.of` factory methods.
- **Assuming Immutability of Input:** Do not pass a vector to the `apply` method assuming it will not be changed. The method's contract is to perform an in-place mutation for performance reasons. If you need to preserve the original vector, clone it first.

## Data Pipeline
Translate acts as a pure processing node in a data flow. It takes a coordinate vector, applies a function, and the mutated vector continues to the next stage.

> Flow:
> Builder Tool Command -> **Translate.of(...)** -> `apply(Vector3i)` -> Modified Voxel/Entity Position -> World State Update

