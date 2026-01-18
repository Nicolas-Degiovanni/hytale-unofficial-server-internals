---
description: Architectural reference for BlockFlipType
---

# BlockFlipType

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Utility Enum

## Definition
```java
// Signature
public enum BlockFlipType
```

## Architecture & Concepts
BlockFlipType is a configuration enum that defines the fundamental geometric behavior of a block when it is subjected to a flip or mirror transformation. It is not a service or a manager, but rather a static data contract that encapsulates transformation logic.

This enum plays a critical role in the block asset pipeline by decoupling the *definition* of a block's flip behavior from the *implementation* of the transformation. A block's asset definition (e.g., in JSON or another format) will specify either ORTHOGONAL or SYMMETRIC. The world engine or block manipulation systems then consume this value to correctly calculate the block's new orientation without needing to hardcode the logic for each block type.

- **ORTHOGONAL:** Represents a 90-degree rotational flip. This is common for blocks that have distinct faces which must map to adjacent faces when flipped, like stairs.
- **SYMMETRIC:** Represents a 180-degree rotational flip. This is used for blocks that are symmetrical along an axis, like slabs or pillars.

This component is a core primitive in the game's geometry and world-building systems.

## Lifecycle & Ownership
- **Creation:** Instances of this enum (ORTHOGONAL, SYMMETRIC) are constructed by the Java Virtual Machine (JVM) during class loading. There is no manual instantiation.
- **Scope:** The enum instances are static and persist for the entire lifetime of the server application. They are effectively global, immutable singletons.
- **Destruction:** Cleanup is handled by the JVM during application shutdown.

## Internal State & Concurrency
- **State:** BlockFlipType is stateless and immutable. Its instances are constants with no mutable fields.
- **Thread Safety:** This enum is inherently thread-safe. Its methods can be called from any thread without synchronization, as they are pure functions that operate only on their inputs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| flipYaw(Rotation rotation, Axis axis) | Rotation | O(1) | Calculates the new Rotation of a block after being flipped along a given Axis. The logic is determined by the specific enum constant (ORTHOGONAL or SYMMETRIC). Returns the existing rotation if the flip axis is Y. |

## Integration Patterns

### Standard Usage
This enum is intended to be retrieved from a block's asset definition and used by systems that manipulate block states, such as a world editor or procedural generation logic.

```java
// Assume 'blockData' holds the asset configuration for a block
// Assume 'currentRotation' is the block's current orientation

BlockFlipType flipType = blockData.getFlipType();
Axis flipAxis = Axis.X; // The axis along which the user wants to flip

// Calculate the new rotation using the block's defined flip behavior
Rotation newRotation = flipType.flipYaw(currentRotation, flipAxis);

// Apply the new rotation to the block in the world
world.setBlockRotation(position, newRotation);
```

### Anti-Patterns (Do NOT do this)
- **External Logic:** Do not use a switch statement on BlockFlipType outside of this class to replicate the flip logic. The `flipYaw` method is the single source of truth for this transformation.
- **Misuse for Pitch/Roll:** The `flipYaw` method is explicitly designed for yaw-based rotations around the Y-axis. Using it to reason about pitch or roll transformations will produce incorrect and unpredictable geometric results.

## Data Pipeline
BlockFlipType acts as a logic node in the data flow of block manipulation. It does not process a stream of data but rather transforms a single data point upon request.

> Flow:
> Block Asset Definition -> **BlockFlipType** -> World Editor / Build Tool -> `flipYaw(rotation, axis)` -> New Rotation State -> World Update

