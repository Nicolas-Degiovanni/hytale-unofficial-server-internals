---
description: Architectural reference for RotationTuple
---

# RotationTuple

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Immutable Value Object / Flyweight

## Definition
```java
// Signature
public record RotationTuple(int index, Rotation yaw, Rotation pitch, Rotation roll) {
```

## Architecture & Concepts
The RotationTuple is a performance-critical value object designed to represent the orientation of a block in 3D space. While appearing as a simple data record, its architecture is built upon the **Flyweight design pattern** to eliminate runtime memory allocation for rotation states, which are finite and frequently accessed.

At its core, the system pre-computes and caches every possible combination of yaw, pitch, and roll into a static array, `VALUES`, during class loading. All factory methods, such as `of()` and `get()`, retrieve instances from this shared cache instead of creating new objects. This design choice is crucial for server performance, as it drastically reduces object churn and garbage collection overhead in systems that manage millions of blocks.

The `index` field serves as a packed integer representation of the rotation. This compact form is ideal for efficient network serialization and persistent storage, allowing the full rotation state to be represented by a single integer. The class acts as the translation layer between this compact integer and the mathematical operations needed to apply the rotation to 3D vectors.

## Lifecycle & Ownership
- **Creation:** All 64 possible RotationTuple instances are instantiated exactly once when the class is loaded by the JVM, within a static initializer block. The public API exclusively provides access to these pre-existing instances.
- **Scope:** Application-scoped. The cached instances persist for the entire lifetime of the server process. They are effectively static constants.
- **Destruction:** The cached instances are garbage collected only when the Java ClassLoader unloads the RotationTuple class, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** **Immutable**. As a Java record, all fields are implicitly final. The internal static `VALUES` array is populated once at startup and is treated as a read-only cache thereafter. The object's state cannot be changed after creation.
- **Thread Safety:** **Fully thread-safe and lock-free**. Its guaranteed immutability and the pre-computation of all instances ensure that a RotationTuple can be safely shared, passed, and read by any number of threads without requiring any synchronization mechanisms.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(yaw, pitch, roll) | RotationTuple | O(1) | Retrieves a pre-cached instance for the specified rotation. |
| get(index) | RotationTuple | O(1) | Retrieves a pre-cached instance using its packed integer index. |
| rotate(vector) | Vector3d | O(1) | Applies the tuple's rotation to a vector, returning a new, transformed Vector3d. |
| index(yaw, pitch, roll) | int | O(1) | Calculates the canonical packed integer index for a given rotation combination. |
| getRotation(rotations, pair, rotation) | RotationTuple | O(N) | A utility to cycle through a specific subset of rotations. N is the length of the input array. |

## Integration Patterns

### Standard Usage
The primary pattern is to obtain a canonical, shared instance via a factory method and use it for mathematical transformations or to retrieve its compact index for storage.

```java
// Obtain a pre-cached instance representing a 90-degree yaw
RotationTuple rotation = RotationTuple.of(Rotation.R90, Rotation.None, Rotation.None);

// Apply this rotation to a block's position vector
Vector3d originalPos = new Vector3d(1.0, 0.0, 0.0);
Vector3d rotatedPos = rotation.rotate(originalPos);

// For serialization, store the compact index
int rotationIndex = rotation.index();
// ... send rotationIndex over the network or save to disk
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** Do not attempt to create new instances of RotationTuple using reflection. The system's performance guarantees rely on using the canonical, pre-cached instances provided by the static factory methods.
- **State Serialization:** Do not serialize the entire RotationTuple object. The correct pattern is to transmit or store the compact `index` integer and reconstruct the object on the receiving end using `RotationTuple.get(index)`. This is significantly more efficient.
- **Custom Indexing Logic:** Do not re-implement the indexing logic. Always rely on the provided `index()` and `get()` methods to ensure forward compatibility and correctness. The underlying formula is an internal implementation detail.

## Data Pipeline
RotationTuple is not a data processor but a value object used within larger pipelines. Its primary role is to convert between a compact integer and a usable rotation object.

> Flow:
> Block Data (from disk/network) -> Integer `index` is read -> **RotationTuple.get(index)** -> Game State stores `RotationTuple` reference -> Physics/Render Engine -> **tuple.rotate(vector)** -> Transformed Vertex Data for rendering

