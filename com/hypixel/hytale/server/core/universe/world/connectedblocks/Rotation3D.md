---
description: Architectural reference for Rotation3D
---

# Rotation3D

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks
**Type:** Transient

## Definition
```java
// Signature
@Deprecated(forRemoval = true)
public class Rotation3D {
```

## Architecture & Concepts
Rotation3D is a mutable data structure designed to represent a three-dimensional orientation using Euler angles: yaw, pitch, and roll. Each axis is represented by a discrete Rotation enum, which quantizes angles into 90-degree increments.

This class is a foundational component within the connected blocks system, where it is used to calculate and store the orientation of blocks that must adapt to their neighbors. Its primary function is to serve as a temporary, mutable container for rotation calculations during block placement or world generation logic.

**WARNING:** This class is explicitly deprecated and marked for future removal. Its use is strongly discouraged in any new development. The mutable design, with public fields and in-place modification methods, is a known source of bugs and has been superseded by more robust, likely immutable, orientation systems such as quaternions or dedicated transform classes. The conversion to and from Vector3f in its rotation logic can also introduce precision loss.

## Lifecycle & Ownership
- **Creation:** Instances are created directly via the public constructor `new Rotation3D(...)`. It is a plain data object with no dependencies on a service locator or injection framework.
- **Scope:** The intended scope is short-lived and typically confined to a single method. It is created to perform a specific set of calculations and is expected to be discarded immediately after.
- **Destruction:** As a simple heap-allocated object with no external resources, it is managed entirely by the Java garbage collector. There is no explicit cleanup or destruction method.

## Internal State & Concurrency
- **State:** The internal state of Rotation3D is highly **mutable**. All three of its core fields, rotationYaw, rotationPitch, and rotationRoll, are public and can be modified directly. Methods such as `add`, `subtract`, and `rotateSelfBy` modify the instance in-place rather than returning a new object. This design is prone to aliasing bugs where modifications in one part of the codebase unexpectedly affect another part that holds a reference to the same instance.
- **Thread Safety:** This class is **not thread-safe**. Its mutable state and lack of any synchronization primitives make it completely unsafe for use in a concurrent environment. If an instance is shared between threads, data races and inconsistent state are guaranteed. All access must be externally synchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assign(yaw, pitch, roll) | void | O(1) | Replaces the current rotation values with the provided ones. |
| add(toAdd) | void | O(1) | Performs an in-place addition of another Rotation3D's values. |
| subtract(toSubtract) | void | O(1) | Performs an in-place subtraction of another Rotation3D's values. |
| negate() | void | O(1) | Inverts the rotation by subtracting it from the zero rotation. |
| rotateSelfBy(yaw, pitch, roll) | Rotation3D | O(1) | Rotates the current orientation by the given amounts. Modifies state in-place. |

## Integration Patterns

### Standard Usage
The intended pattern is to use Rotation3D as a temporary calculator for block orientation logic. An instance is created, manipulated through a series of transformations, and its final state is read to configure a game object.

```java
// Example: Calculating a final block orientation
Rotation3D baseRotation = new Rotation3D(Rotation.None, Rotation.None, Rotation.None);
Rotation3D offset = getPlacementOffsetRotation();

// Apply transformations in-place
baseRotation.add(offset);
baseRotation.rotateSelfBy(Rotation.Degrees90, Rotation.None, Rotation.None);

// Use the final values
BlockState.setRotation(baseRotation.rotationYaw, baseRotation.rotationPitch);
```

### Anti-Patterns (Do NOT do this)
- **Persistent State:** Do not store instances of Rotation3D as long-lived state in components or entities. Its mutable nature makes it unsuitable for this purpose.
- **Shared References:** Never pass a Rotation3D instance to multiple systems that may modify it independently. This will lead to unpredictable behavior and race conditions. Create a new instance or a copy instead.
- **Concurrent Access:** Do not read or write to a Rotation3D instance from multiple threads without external locking. The class provides no internal thread safety.
- **New Development:** The most critical anti-pattern is using this class at all in new code, as it is deprecated and will be removed from the engine.

## Data Pipeline
Rotation3D is not a data pipeline component itself but rather a tool used within one. It acts as a transient state object during a calculation phase.

> Flow:
> Initial Block Data -> **new Rotation3D()** -> In-place modifications via `add()`, `rotateSelfBy()` -> Final rotation values read -> World State Update

