---
description: Architectural reference for SensorBlock
---

# SensorBlock

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient

## Definition
```java
// Signature
public class SensorBlock extends SensorBase {
```

## Architecture & Concepts
The SensorBlock is a fundamental component of the Hytale NPC AI system, operating within the "Sense" phase of a Sense-Think-Act cycle. Its primary function is to detect the presence of specific block types within a defined radius of an NPC. This component acts as a conditional trigger for NPC behaviors, enabling actions like resource gathering, pathfinding to points of interest, or environment interaction.

It is not a standalone service but rather a configurable, data-driven object that is part of an NPC's **Role**. A Role defines the overarching behavior of an NPC, and Sensors like SensorBlock provide the perceptual input required for its decision-making logic, such as a state machine or behavior tree.

SensorBlock integrates deeply with two key systems:
1.  **World State:** It directly queries the game `World` to check for blocks, examining `WorldChunk` and `BlockSection` data structures.
2.  **AI Blackboard:** It leverages the `Blackboard` system as a high-performance spatial index for block types (`BlockTypeView`) and as a mechanism for coordinating resource access between multiple NPCs (`ResourceView`). This prevents multiple NPCs from targeting the same block simultaneously if it is configured as a unique resource.

A critical optimization pattern is its use of a `BlockTarget` cache, managed by the parent `Role`. This cache stores the last known valid block, preventing expensive world scans on every single game tick. The sensor is responsible for both validating this cache and repopulating it when it becomes stale.

## Lifecycle & Ownership
-   **Creation:** SensorBlock instances are not created dynamically during gameplay. They are instantiated once during server startup or asset loading by a corresponding `BuilderSensorBlock`. This process parses NPC configuration files (e.g., JSON or HOCON) and constructs the sensor as part of a larger, immutable `Role` definition.
-   **Scope:** The object's lifetime is tied to the `Role` asset it belongs to. It is effectively a stateless configuration object that is shared and reused by all NPC entities assigned that specific Role. It persists for the entire server session unless assets are reloaded.
-   **Destruction:** The object is marked for garbage collection only when its parent `Role` asset is unloaded from memory, typically during a server shutdown or a full asset reload.

## Internal State & Concurrency
-   **State:** The core configuration of a SensorBlock (`range`, `blockSet`, etc.) is immutable, established at creation time and stored in `final` fields. However, the component is not pure; its `matches` method produces side effects by updating a cached `BlockTarget` on the NPC's `Role` and modifying its internal `positionProvider`. The `positionProvider` field holds the result of the last sensor evaluation and is cleared at the start of each check, making the component's state transient on a per-call basis.

-   **Thread Safety:** **This component is not thread-safe.** It is designed to be executed exclusively by the thread responsible for updating a specific NPC entity. The `matches` method reads from and writes to shared, mutable data structures like the `World` and the `Blackboard`. The underlying systems are expected to manage their own concurrency, but the SensorBlock itself contains no internal locking.

    **WARNING:** Invoking the `matches` method for the same SensorBlock instance from multiple threads concurrently will lead to race conditions, particularly with the `positionProvider` state and the external `BlockTarget` cache. All sensor evaluations must be synchronized with the main entity update loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(N) | Evaluates the world state. Returns true if a matching block is found within range. Complexity is proportional to the number of blocks (N) in the search volume. |
| getSensorInfo() | InfoProvider | O(1) | Returns the provider holding the results of the last `matches` call, primarily the target position. |

## Integration Patterns

### Standard Usage
A developer does not interact with SensorBlock directly in code. It is defined declaratively in an NPC's asset files. The game's AI engine invokes the `matches` method as part of a behavior tree or state machine evaluation. A subsequent action node would then use `getSensorInfo` to retrieve the target location.

```java
// Engine-level code (conceptual)
// Within an NPC's update tick:

// The engine checks if the sensor's condition is met
boolean blockFound = currentBehavior.getSensor().matches(npcRef, role, dt, store);

if (blockFound) {
    // An action component retrieves the location found by the sensor
    PositionProvider posProvider = (PositionProvider) currentBehavior.getSensor().getSensorInfo();
    Vector3d targetPosition = posProvider.getTarget();

    // The action component uses the position (e.g., to move the NPC)
    moveToComponent.setDestination(targetPosition);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new SensorBlock()`. The object is complex to configure and requires a builder pattern tied to the asset loading pipeline. Direct instantiation will result in a non-functional sensor.
-   **State Persistence:** Do not assume the `PositionProvider` returned by `getSensorInfo` is valid across multiple ticks. Its state is ephemeral and is only guaranteed to be correct immediately after a `matches` call that returned true.
-   **Ignoring Blackboard Dependencies:** Attempting to use a SensorBlock without the required `BlockTypeView` being populated on the `Blackboard` will cause the sensor to fail its search. The constructor explicitly declares this dependency.

## Data Pipeline
The flow of data for a block detection query is a multi-stage process involving caching, world queries, and resource management.

> Flow:
> NPC Behavior Tick -> `SensorBlock.matches()` -> Check `Role` for cached `BlockTarget` -> **(Cache Hit)** -> Validate block still exists -> Return `true`
>
> **(Cache Miss or Invalid)** -> Query `Blackboard (BlockTypeView)` for nearest block -> **SensorBlock** -> If found, update `BlockTarget` cache -> Optionally reserve block via `Blackboard (ResourceView)` -> Populate internal `PositionProvider` -> Return `true`
>
> **(No Block Found)** -> Clear `PositionProvider` -> Return `false`

