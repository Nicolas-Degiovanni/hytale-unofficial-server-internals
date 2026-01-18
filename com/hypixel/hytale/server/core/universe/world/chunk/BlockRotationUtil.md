---
description: Architectural reference for BlockRotationUtil
---

# BlockRotationUtil

**Package:** com.hypixel.hytale.server.core.universe.world.chunk
**Type:** Utility

## Definition
```java
// Signature
public class BlockRotationUtil {
```

## Architecture & Concepts
BlockRotationUtil is a stateless, pure-functional utility class that provides the mathematical foundation for all block orientation transformations within the server. It acts as a centralized calculation engine for rotating and flipping blocks, ensuring that these complex geometric operations are handled consistently across all high-level systems.

Its primary role is to decouple the intricate logic of 3D rotation mathematics from systems like world generation, structure placement (prefabs), and player interaction. Instead of embedding rotation calculations within these systems, they delegate the task to this utility. The class operates on two distinct representations of block orientation:
1.  **RotationTuple:** A high-level object representing yaw, pitch, and roll.
2.  **Filler:** A primitive integer likely used as a compact, bit-packed representation of block state for performance-critical operations like chunk serialization.

By providing a static, globally accessible interface, BlockRotationUtil guarantees that any part of the server needing to transform a block's orientation does so using the exact same, verified algorithms.

### Lifecycle & Ownership
-   **Creation:** This class is never instantiated. As a utility class containing only static methods, its functionality is accessed directly via the class reference.
-   **Scope:** The methods of BlockRotationUtil are available for the entire lifetime of the server application, once the class is loaded by the JVM. It has no instance-based scope.
-   **Destruction:** Not applicable. The class definition is unloaded when its ClassLoader is garbage collected, typically during server shutdown.

## Internal State & Concurrency
-   **State:** BlockRotationUtil is completely stateless. It contains no member fields and all its methods are pure functions; their output depends exclusively on their input arguments. No data is cached or stored between calls.
-   **Thread Safety:** This class is inherently and unconditionally thread-safe. Its stateless design eliminates any possibility of race conditions or data corruption from concurrent access. It can be safely invoked from any thread, including the main server thread, world generation worker threads, or network I/O threads, without requiring any external synchronization or locks.

## API Surface
The public API provides a set of functions for transforming block orientations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFlipped(blockRotation, flipType, axis, variantRotation) | RotationTuple | O(1) | Calculates the new orientation of a block after being flipped along a specified axis. Returns null if the resulting orientation is invalid for the given VariantRotation. |
| getRotated(blockRotation, axis, rotation, variantRotation) | RotationTuple | O(1) | Calculates the new orientation of a block after being rotated around a specified axis. Returns null if the resulting orientation is invalid. |
| getFlippedFiller(filler, axis) | int | O(1) | Calculates the new integer filler value for a block flipped 180 degrees along an axis. This is a low-level, high-performance alternative to the RotationTuple variant. |
| getRotatedFiller(filler, axis, rotation) | int | O(1) | Calculates the new integer filler value for a block rotated around an axis. |

## Integration Patterns

### Standard Usage
This class should always be used statically. It is a common dependency for any system that programmatically places or modifies blocks in the world, such as a structure or prefab placement engine.

```java
// Example: Rotating a block's orientation for a prefab
RotationTuple currentOrientation = ...;
VariantRotation blockVariant = ...;

// Rotate the block 90 degrees around the Y-axis
RotationTuple newOrientation = BlockRotationUtil.getRotated(
    currentOrientation,
    Axis.Y,
    Rotation.Ninety,
    blockVariant
);

if (newOrientation != null) {
    // Apply the new orientation to the block in the world
}
```

### Anti-Patterns (Do NOT do this)
-   **Attempted Instantiation:** The class has no public constructor and holds no state. Attempting to create an instance is incorrect and serves no purpose.
-   **Logic Duplication:** Do not re-implement block rotation logic elsewhere in the codebase. This central utility exists to prevent inconsistencies and bugs arising from disparate, competing implementations.
-   **Ignoring Null Returns:** The methods operating on RotationTuple can return null to signify an invalid or unsupported transformation for a specific block type. Failure to check for null can lead to NullPointerExceptions in world generation or block placement code.

## Data Pipeline
BlockRotationUtil is not a stage in a data pipeline itself, but rather a functional component invoked by various pipeline stages. It acts as a pure transformation function on block state data as it flows through other systems.

> **World Generation Flow:**
> Prefab Asset Loader -> Structure Placement Service -> **BlockRotationUtil**(calculates orientation for each block) -> Chunk Data Buffer -> World Persistence

> **Player Interaction Flow:**
> Network Packet (Player Places Block) -> Block Placement Logic -> **BlockRotationUtil**(calculates final orientation based on player facing) -> World State Update -> Network Packet (Broadcast Block Update)

