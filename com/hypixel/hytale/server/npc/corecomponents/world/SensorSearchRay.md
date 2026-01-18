---
description: Architectural reference for SensorSearchRay
---

# SensorSearchRay

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorSearchRay extends SensorBase {
```

## Architecture & Concepts
The SensorSearchRay is a specialized environmental sensor within the Non-Player Character (NPC) AI framework. Its primary function is to detect the presence of specific block types within an NPC's forward-facing field of view. It operates by performing a constrained raycast originating from the NPC's head position.

This component is a fundamental building block for creating complex, world-aware NPC behaviors, such as resource gathering, obstacle identification, or searching for points of interest.

Architecturally, it is designed for high performance in a dense server environment. It employs several layers of caching and throttling to avoid executing expensive raycast operations on every server tick. This optimization is critical for server performance, as a large number of NPCs could otherwise degrade world simulation. The sensor's logic prioritizes cache validation and state-change thresholds before committing to a new world query.

## Lifecycle & Ownership
- **Creation:** An instance of SensorSearchRay is created by the server's NPC asset pipeline when an NPC entity is spawned. Its configuration is defined declaratively in an NPC asset file and instantiated through its corresponding builder, BuilderSensorSearchRay. It is not intended for manual instantiation.
- **Scope:** The lifecycle of a SensorSearchRay instance is tightly coupled to its parent NPC entity. It persists as a component of the NPC's Role for the entire duration the NPC is active in the world.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection when its parent NPC entity is unloaded or destroyed. No explicit cleanup method is required.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. It maintains internal state about the last check time, the NPC's position and orientation during the last check, and the revision of the world chunk section that was hit. This state is essential for its throttling and caching mechanisms. The result of a successful scan is stored in the PositionProvider.

- **Thread Safety:** **This class is not thread-safe and must not be accessed concurrently.** It is designed to be operated exclusively by the single thread responsible for ticking the parent NPC's AI. The use of a ThreadLocal instance for RayBlockHitTest is a strong indicator of this single-threaded design assumption. Unsynchronized, concurrent calls to the matches method will lead to race conditions and unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | Variable (O(1) to O(N)) | The primary update method. Returns true if a target block is detected. Complexity is O(1) on a cache hit and O(N) on a cache miss, where N is the number of blocks traversed by the ray. |
| getSensorInfo() | InfoProvider | O(1) | Retrieves the result of the last successful sensor check. The returned PositionProvider contains the coordinates of the detected block. |

## Integration Patterns

### Standard Usage
This sensor is not invoked directly. It is configured as part of an NPC's definition and is automatically processed by the server's AI behavior system. The system calls the matches method each tick to evaluate the sensor's state, which can then be used as a condition to trigger transitions in a behavior tree.

```java
// Conceptual usage within an NPC Behavior Tree Node
// Note: This is a simplified representation. Developers do not write this code directly.

boolean blockInSight = npc.getSensor(SensorSearchRay.class).matches(ref, role, dt, store);

if (blockInSight) {
    InfoProvider info = npc.getSensor(SensorSearchRay.class).getSensorInfo();
    Vector3d targetPosition = ((PositionProvider) info).getTarget();
    // Trigger "MoveTo" behavior with targetPosition
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using a constructor. Sensors must be defined in NPC asset files and managed by the engine's lifecycle systems.
- **Concurrent Access:** Do not call any methods on this class from any thread other than the NPC's primary update thread. This will corrupt its internal state and the shared ThreadLocal raycast utility.
- **Stale Data Retrieval:** Do not call getSensorInfo without first calling matches in the same tick and verifying it returned true. The data in the PositionProvider is only valid immediately after a successful match.

## Data Pipeline
The data flow for this sensor involves querying the world state based on the NPC's current transform and returning a positional result.

> Flow:
> NPC AI Tick -> **SensorSearchRay.matches()** -> Cache & Throttle Check -> RayBlockHitTest (ThreadLocal) -> World Chunk Data -> **SensorSearchRay.positionProvider** -> NPC Behavior Tree Condition

