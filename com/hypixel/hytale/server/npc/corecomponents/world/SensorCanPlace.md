---
description: Architectural reference for SensorCanPlace
---

# SensorCanPlace

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient

## Definition
```java
// Signature
public class SensorCanPlace extends SensorBase {
```

## Architecture & Concepts
The SensorCanPlace component is a perceptual predicate within the server-side NPC AI framework. It does not execute actions but instead evaluates a specific world-state condition: whether an NPC can place a designated block at a location relative to its current position and orientation.

This component acts as a gatekeeper for construction-related behaviors. In architectures like Behavior Trees, it would serve as a *Condition* node. An AI task, such as "Build Wall", would first query this sensor. Only if SensorCanPlace returns true will the AI proceed to an *Action* node that actually places the block.

Its core function involves complex geometric calculations to determine a valid target block coordinate. It accounts for the NPC's bounding box, the target block's bounding box, and the NPC's yaw to find an adjacent, non-obstructed placement location. To mitigate performance costs associated with frequent world queries, it employs a caching and throttling mechanism.

## Lifecycle & Ownership
- **Creation:** Instantiated by the NPC AI factory during the assembly of an NPC's behavior set. Its configuration, such as direction and offset, is provided by a corresponding BuilderSensorCanPlace instance, which is typically loaded from NPC asset definitions.
- **Scope:** The lifetime of a SensorCanPlace instance is tied directly to its parent NPC. It persists as long as the NPC exists and its AI configuration is active. Each NPC with this sensor capability has its own unique instance.
- **Destruction:** The object is marked for garbage collection when the parent NPC entity is unloaded or destroyed. The `clearOnce` method provides a mechanism for manually resetting its internal timer state without requiring full re-instantiation.

## Internal State & Concurrency
- **State:** Highly mutable. The component maintains internal state for performance optimization, including the last calculated target position (cachedPosition), the last boolean result (cachedResult), and a time-based delay (delay). This stateful design is critical to prevent re-running expensive world queries on every server tick.
- **Thread Safety:** **WARNING:** This class is not thread-safe. It is designed to be accessed exclusively from the main server thread responsible for NPC updates. Its mutable state and lack of synchronization primitives make it inherently unsafe for concurrent access. Modifying or reading its state from other threads will lead to race conditions and unpredictable AI behavior.

## API Surface
The public contract is focused on evaluation and data retrieval for subsequent AI components.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | Amortized O(1) | Evaluates if the NPC can place its currently designated block. Returns a cached result if called within the retryDelay window. Triggers expensive world queries only when the cache is stale. |
| getSensorInfo() | InfoProvider | O(1) | Retrieves the data provider that exposes the calculated target placement position. **WARNING:** This data is only valid after `matches` returns true. |
| clearOnce() | void | O(1) | Resets the internal retry delay timer, forcing a full re-evaluation on the next call to `matches`. |

## Integration Patterns

### Standard Usage
This component is designed to be used as a conditional check within a higher-level AI behavior loop. The boolean result gates further actions, and the InfoProvider supplies the necessary positional data to those actions.

```java
// Inside an NPC's AI update tick
SensorCanPlace placementSensor = npc.getSensor(SensorCanPlace.class);

// First, check the condition
if (placementSensor.matches(npcRef, npcRole, deltaTime, entityStore)) {
    // If successful, retrieve the target position from the sensor's info provider
    InfoProvider info = placementSensor.getSensorInfo();
    Vector3d targetPosition = ((CachedPositionProvider) info).getTarget();

    // Now, enqueue an action to place the block at the validated position
    npc.getBrain().enqueueAction(new PlaceBlockAction(targetPosition));
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SensorCanPlace()`. The component relies on a builder for proper configuration of its direction, offset, and retry delay. It must be created by the server's NPC asset loading and AI assembly pipeline.
- **State Polling:** Do not call `getSensorInfo` to check for a valid position without first calling `matches` in the same tick. The InfoProvider is only populated with valid data after a successful `matches` evaluation and is cleared upon failure. Polling it directly can lead to using stale data from a previous, successful tick.
- **Ignoring The Return Value:** The boolean return from `matches` is the canonical signal of success. Relying solely on the state of `getSensorInfo` is an error-prone pattern. If `matches` returns false, the data in the provider is invalid.

## Data Pipeline
The component transforms NPC and World state into a validated placement position for use by other AI systems.

> Flow:
> NPC State (Transform, Role) -> **SensorCanPlace.matches()** -> BlockPlacementHelper -> World Voxel & Entity Data -> Cached Internal State -> InfoProvider -> AI Action Component

