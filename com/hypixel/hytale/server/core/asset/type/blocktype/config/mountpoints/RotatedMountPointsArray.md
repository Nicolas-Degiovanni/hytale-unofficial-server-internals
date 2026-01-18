---
description: Architectural reference for RotatedMountPointsArray
---

# RotatedMountPointsArray

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config.mountpoints
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class RotatedMountPointsArray {
```

## Architecture & Concepts
The RotatedMountPointsArray is a specialized data structure that encapsulates the mount points defined for a single block type. Its primary architectural purpose is to provide an efficient, on-demand calculation and caching mechanism for the spatial orientation of these points as a block is rotated in the game world.

This class is not a service or manager; it is a value object, typically a field within a larger block configuration asset. It is designed to be instantiated directly from asset data files (e.g., JSON) via the Hytale serialization framework, a fact evidenced by its public static CODEC field.

The core design pattern employed is **lazy evaluation with memoization**. Rotated versions of the mount points are not pre-calculated when the object is created. Instead, they are computed only when first requested for a specific rotation index. The result of this expensive computation is then cached internally, ensuring that subsequent requests for the same rotation are served instantly from memory. This strategy optimizes for both memory usage (by not storing all possible rotations) and performance (by avoiding redundant calculations).

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale Codec system during the deserialization of a block asset. The static CODEC field acts as the factory, constructing an object from a raw array of BlockMountPoint data within the asset file. Direct instantiation by game logic is an anti-pattern.
-   **Scope:** The lifetime of a RotatedMountPointsArray instance is tightly coupled to its parent block type configuration object. It persists in memory as long as the corresponding block asset is loaded.
-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection when its parent block type asset is unloaded by the engine, for example, when transitioning between worlds or shutting down a server. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** The object's state is mutable. It is initialized with an immutable `raw` array of mount points. The `rotated` field, a two-dimensional array, acts as a mutable, lazily-populated cache. The `transient` keyword on this field is a critical indicator that this cache is runtime-only and is not part of the object's serialized form.

-   **Thread Safety:** **This class is not thread-safe.** The lazy initialization logic within the getRotated method contains a race condition. If multiple threads attempt to access a previously uncalculated rotation simultaneously, they may both enter the cache-population block. This can lead to redundant computation and potentially inconsistent state. All access from multi-threaded contexts, such as a parallelized chunk generation system, **must** be externally synchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| size() | int | O(1) | Returns the number of base, non-rotated mount points. |
| getRaw() | BlockMountPoint[] | O(1) | Returns the original array of mount points as defined in the asset file. |
| getRotated(int rotationIndex) | BlockMountPoint[] | O(N) / O(1) | Returns an array of mount points for the given rotation. Complexity is O(N) on the first call for a specific index and O(1) on all subsequent calls for that same index due to caching. |

## Integration Patterns

### Standard Usage
This object is intended to be retrieved from a loaded block configuration. Game systems then use the block's current rotation state in the world to query for the correctly oriented mount points.

```java
// Assume 'blockConfig' is the loaded asset for a specific block type
RotatedMountPointsArray mountPoints = blockConfig.getMountPoints();

// 'currentState' would be retrieved from the world for a block at a given position
int currentRotationIndex = currentState.getRotationIndex();

// Retrieve the correctly oriented mount points. This may trigger a one-time calculation.
BlockMountPoint[] activeMounts = mountPoints.getRotated(currentRotationIndex);

if (activeMounts != null) {
    for (BlockMountPoint mount : activeMounts) {
        // Use the mount point for entity attachment, particle effects, etc.
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new RotatedMountPointsArray()`. The object is fundamentally tied to the asset loading pipeline and will be uninitialized and useless if created manually.
-   **Modification of Returned Arrays:** The arrays returned by getRaw and getRotated are direct references to the internal state and cache. Modifying the contents of these arrays will permanently corrupt the block type's definition for the entire session, leading to severe and hard-to-diagnose visual and logical bugs. Treat all returned arrays as read-only.
-   **Unsynchronized Multi-threaded Access:** As detailed in the Concurrency section, calling getRotated from multiple threads without an external lock is unsafe and will lead to race conditions.

## Data Pipeline
The flow of data originates from static asset files and is transformed into an interactive, cached object used by live game systems.

> Flow:
> Block Asset File (JSON) -> Hytale Codec Deserializer -> **RotatedMountPointsArray** (Instance with `raw` data) -> Game System Request -> On-Demand Rotation Calculation -> Internal `rotated` Cache -> Final `BlockMountPoint[]` used by Logic

