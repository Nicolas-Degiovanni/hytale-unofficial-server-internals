---
description: Architectural reference for PositionProbeBase
---

# PositionProbeBase

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Transient Utility

## Definition
```java
// Signature
public class PositionProbeBase {
```

## Architecture & Concepts

The PositionProbeBase is a stateful utility class designed to perform environmental queries at a specific point in the world. It serves as a foundational tool for higher-level systems, particularly Non-Player Character (NPC) artificial intelligence, pathfinding, and custom physics logic.

Its primary function is to abstract the complexity of querying various low-level world data structures. Instead of a system needing to manually interact with the ChunkStore, CollisionModule, and BlockChunk components, it can use a PositionProbe to gather a comprehensive snapshot of a location's properties in a single operation. The probe determines collision validity, ground and water levels, and an entity's relationship to those surfaces (e.g., on ground, in water, height above ground).

This class is designed as a base for more specialized probes. It is not a managed service but a transient "tool" object, created on-demand to perform a query and then discarded.

## Lifecycle & Ownership

-   **Creation:** An instance of PositionProbeBase (or a subclass) is created directly via its constructor, typically at the beginning of a task that requires world-space awareness, such as an AI behavior's update cycle. It is not managed by a dependency injection framework or a service locator.
-   **Scope:** The intended scope is extremely short-lived, usually confined to a single method or a single game tick. An instance holds the state of exactly one probe operation.
-   **Destruction:** The object is managed by the standard Java Garbage Collector. Once the reference to the probe goes out of scope, it becomes eligible for collection. There are no native resources or explicit cleanup methods to call.

## Internal State & Concurrency

-   **State:** PositionProbeBase is highly mutable. Its core purpose is to populate its own instance fields based on the results of the `probePosition` method. The object effectively serves as a result container for its own primary operation. It maintains a small internal cache for the last water level check to prevent redundant, expensive lookups when probing nearby positions sequentially.
-   **Thread Safety:** This class is **not thread-safe**. All primary operations mutate shared instance fields without any synchronization. Sharing a single instance across multiple threads will result in race conditions and a corrupted, unpredictable state. Each thread of execution requiring a position probe must create and use its own unique instance.

## API Surface

The public API consists primarily of getters that expose the state calculated by the protected `probePosition` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| probePosition(...) | protected boolean | O(C) | The core operation. Populates all internal state fields by querying world data. C is the complexity of the underlying collision check. Returns true if the position is in a loaded chunk. |
| isValidPosition() | boolean | O(1) | Returns true if the associated bounding box does not collide with any solid blocks at the probed position. |
| isOnGround() | boolean | O(1) | Returns true if the probe determined the entity is on the ground. State is set by a subclass, not this base class. |
| isInWater() | boolean | O(1) | Returns true if the probe determined the entity is in water. State is set by a subclass. |
| getGroundLevel() | int | O(1) | Returns the Y-coordinate of the highest solid block in the column of the probed position. Returns -1 if not in a loaded chunk. |
| getWaterLevel() | int | O(1) | Returns the Y-coordinate of the water surface in the column of the probed position. Returns -1 if not in a loaded chunk. |
| getHeightOverGround() | int | O(1) | Returns the vertical distance in blocks between the entity's position and the ground. |
| getDepthBelowSurface() | int | O(1) | Returns the vertical distance in blocks the entity is below the water surface. |

## Integration Patterns

### Standard Usage

PositionProbeBase is intended to be extended. A consumer would instantiate a subclass, execute a probe, and then use the getters to inform its logic.

```java
// A hypothetical AI behavior using a specialized probe
// MyPositionProbe extends PositionProbeBase

MyPositionProbe probe = new MyPositionProbe();
Vector3d targetPosition = new Vector3d(100, 64, 200);
Box boundingBox = entity.getBoundingBox();

// The probe is executed, populating its internal state
boolean isChunkLoaded = probe.probePosition(entityRef, boundingBox, targetPosition, ...);

if (isChunkLoaded && probe.isValidPosition()) {
    if (probe.getGroundLevel() > 60) {
        // Pathfind towards this valid, high-ground position
    } else if (probe.isInWater()) {
        // Switch to swimming behavior
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not re-use a probe instance across multiple, unrelated game ticks or for different entities without a deep understanding of its state. The internal state, including the water level cache, can become stale.
-   **Concurrent Access:** **CRITICAL:** Never share a PositionProbeBase instance across threads. Each thread must create its own instance to avoid data corruption.
-   **Treating as a Service:** This is a simple stateful object, not a managed service. Do not attempt to register it with or retrieve it from a central service locator. Its lifecycle is meant to be controlled manually and kept short.

## Data Pipeline

The `probePosition` method orchestrates a flow of data from several core server modules to populate its internal state.

> Flow:
> (Vector3d, Box) -> ChunkStore Lookup -> CollisionModule.validatePosition -> BlockChunk.getHeight -> WorldUtil.getWaterLevel -> **PositionProbeBase** (Internal state is populated) -> Consumer calls get...() methods to read results

