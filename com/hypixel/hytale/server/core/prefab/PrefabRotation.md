---
description: Architectural reference for PrefabRotation
---

# PrefabRotation

**Package:** com.hypixel.hytale.server.core.prefab
**Type:** Utility

## Definition
```java
// Signature
public enum PrefabRotation {
```

## Architecture & Concepts
PrefabRotation is a foundational utility enum that encapsulates the four cardinal rotations around the vertical Y-axis (0, 90, 180, and 270 degrees). It is a critical component in the world generation and structure placement systems, responsible for orienting prefabs and their constituent blocks correctly within the game world.

Architecturally, this enum employs the **Strategy Pattern**. Each enum constant (ROTATION_0, ROTATION_90, etc.) holds a private, final instance of a corresponding RotationExecutor implementation. The public methods of PrefabRotation, such as *rotate* or *getX*, delegate the actual mathematical transformation to the specific executor instance. This design elegantly avoids conditional branching (switch or if/else statements) within the core rotation logic, leading to clean, highly performant, and predictable code.

Its primary role is to act as a transformer for spatial data. It operates on vector coordinates (Vector3d, Vector3i, Vector3l) and encoded block state data, ensuring that when a pre-designed structure is placed, all its parts are rotated consistently.

### Lifecycle & Ownership
- **Creation:** The four instances of PrefabRotation are created and initialized by the JVM during class loading. As an enum, its lifecycle is managed entirely by the runtime.
- **Scope:** These instances are static, global, and persist for the entire lifetime of the server application.
- **Destruction:** The instances are destroyed only when the JVM shuts down.

## Internal State & Concurrency
- **State:** PrefabRotation is **completely immutable**. Its internal fields, *rotation* and *executor*, are final and assigned at construction. The enum itself holds no mutable state.
- **Thread Safety:** Due to its immutable nature, PrefabRotation is inherently **thread-safe**. It can be safely used and shared across multiple threads, such as parallel world generation workers, without any need for external synchronization or locks.

## API Surface
The API provides methods for transforming coordinates and block data. Note that vector rotation methods mutate the input vector instance.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromRotation(Rotation) | PrefabRotation | O(1) | Static factory method to convert from the lower-level Rotation enum. |
| valueOfExtended(String) | PrefabRotation | O(1) | Static factory method for string parsing, supporting prefixed and non-prefixed names. |
| add(PrefabRotation) | PrefabRotation | O(1) | Composes this rotation with another, returning the resulting PrefabRotation. |
| rotate(Vector3d/i/l v) | void | O(1) | **Mutates** the provided vector, rotating its X and Z components in-place. |
| getX(int x, int z) | int | O(1) | Calculates the new X coordinate after rotation without modifying any state. |
| getZ(int x, int z) | int | O(1) | Calculates the new Z coordinate after rotation without modifying any state. |
| getYaw() | float | O(1) | Returns the equivalent yaw rotation in radians. |
| getRotation(int) | int | O(1) | Transforms an encoded block rotation value. |
| getFiller(int) | int | O(1) | Transforms an encoded block "filler" value, which may contain orientation data. |

## Integration Patterns

### Standard Usage
PrefabRotation is used to orient coordinates during structure placement. A rotation is selected and then applied to each point within the prefab's local space to find its final world space position.

```java
// Example: Rotate a local prefab coordinate into its world position
Vector3i localPoint = new Vector3i(5, 10, 2);
PrefabRotation structureRotation = PrefabRotation.ROTATION_270;

// The rotate method modifies the vector in-place
structureRotation.rotate(localPoint);

// localPoint is now (-2, 10, 5)
// This new vector can now be added to the prefab's origin in world space.
```

### Anti-Patterns (Do NOT do this)
- **Manual Rotation Logic:** Do not write a switch statement based on the enum's value to perform rotation. The entire purpose of the class's design is to encapsulate this logic. Use the provided *rotate*, *getX*, and *getZ* methods.
- **Ignoring Vector Mutation:** The *rotate(Vector3...)* methods modify the vector instance they are given. If the original, un-rotated vector is needed later, you must clone it before passing it to the rotate method. Failure to do so is a common source of bugs.
    ```java
    // BAD: originalPoint is lost
    Vector3i originalPoint = new Vector3i(10, 0, 0);
    rotation.rotate(originalPoint);
    // originalPoint is now changed forever

    // GOOD: Preserving the original vector
    Vector3i originalPoint = new Vector3i(10, 0, 0);
    Vector3i rotatedPoint = originalPoint.clone();
    rotation.rotate(rotatedPoint);
    ```

## Data Pipeline
PrefabRotation is not a pipeline stage itself, but rather a stateless operator applied to data as it flows through a pipeline, typically during world generation. It transforms data from one coordinate system to another.

> Flow:
> Prefab Asset (Local Coordinates) -> World Generator -> **PrefabRotation Operator** -> Final Chunk Data (World Coordinates)

