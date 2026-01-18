---
description: Architectural reference for VoxelCoordinate
---

# VoxelCoordinate

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures.voxelspace
**Type:** Transient

## Definition
```java
// Signature
public class VoxelCoordinate {
```

## Architecture & Concepts
The VoxelCoordinate class is a fundamental data structure representing a discrete, integer-based point within a three-dimensional grid. It serves as the primary addressing mechanism for all voxel-based systems, including world generation, physics simulation, block manipulation, and AI pathfinding.

Conceptually, it is a lightweight value object whose sole purpose is to encapsulate X, Y, and Z integer coordinates. Its existence provides type safety and semantic clarity, preventing the error-prone practice of passing raw integer arrays or multiple integer arguments to represent a spatial position. While originating in the world generation package, its use is pervasive across any engine component that needs to reason about the game world at the block level.

## Lifecycle & Ownership
- **Creation:** VoxelCoordinate instances are created on-demand by any system requiring a spatial reference. Due to its fundamental role, this class is subject to extremely high allocation frequency, particularly within tight loops in world generation or physics update cycles.
- **Scope:** The lifetime of a VoxelCoordinate is typically ephemeral. Most instances exist only within the scope of a single method call or a single frame update. They are frequently used as temporary variables or passed as parameters. In some cases, they may be stored as keys in longer-lived collections, such as a map of special block locations.
- **Destruction:** As a standard heap-allocated object with no special resource handles, instances are managed entirely by the Java Garbage Collector. They become eligible for collection as soon as they are no longer reachable.

**WARNING:** The high allocation rate of this class can exert significant pressure on the garbage collector. Performance-critical systems should consider object pooling or reusing VoxelCoordinate instances where safe to do so.

## Internal State & Concurrency
- **State:** The internal state is **Mutable**. The x, y, and z fields are not declared as final and can be modified after instantiation. This design choice allows for object reuse to reduce allocations, but introduces significant risks if not managed carefully.
- **Thread Safety:** This class is **Not Thread-Safe**. There are no synchronization mechanisms, locks, or volatile keywords. Concurrent access from multiple threads, especially write operations, will result in data races, memory visibility issues, and undefined behavior. Any multi-threaded system using VoxelCoordinate must implement its own external locking or ensure that instances are confined to a single thread.

**WARNING:** Sharing a mutable VoxelCoordinate instance between threads or systems without proper synchronization is a severe programming error and will lead to difficult-to-diagnose bugs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| VoxelCoordinate(x, y, z) | constructor | O(1) | Creates a new coordinate instance. |
| equals(other) | boolean | O(1) | Performs a value-based equality check. |
| clone() | VoxelCoordinate | O(1) | Creates and returns a new VoxelCoordinate instance with identical values. |
| toString() | String | O(1) | Provides a string representation for debugging. |

## Integration Patterns

### Standard Usage
The intended use is as a transient data-holder to pass coordinate information between methods or systems. When a safe, independent copy is needed, the clone method should be used.

```java
// Standard creation and use as a parameter
VoxelCoordinate target = new VoxelCoordinate(10, 64, -50);
World.setBlock(target, Block.STONE);

// Using clone() to prevent unintended side-effects
VoxelCoordinate playerPos = player.getVoxelCoordinate();
VoxelCoordinate safeCopy = playerPos.clone();
Pathfinder.calculatePathTo(safeCopy);
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** Passing a single VoxelCoordinate instance to multiple subsystems that may modify it is extremely dangerous. One system's change will silently affect the other, leading to "spooky action at a distance".

- **Missing hashCode Implementation:** The class correctly implements equals but **fails to override hashCode**. This violates the Java contract for these two methods. Using VoxelCoordinate as a key in a HashMap or an element in a HashSet will result in incorrect behavior, as the default identity-based hash code will be used. Two separate instances with the same X, Y, Z values will produce different hash codes and will not be treated as equivalent keys.

**CRITICAL:** Do not use this class as a key in hash-based collections until a proper `hashCode` method is implemented.

## Data Pipeline
VoxelCoordinate is not a processing stage in a pipeline; rather, it is the data that flows through it. It often represents the input or output of a spatial query or world modification operation.

> Flow:
> Player Input -> Raycast System -> **VoxelCoordinate** (target block) -> WorldManipulationService -> Network Packet (Block Update)

