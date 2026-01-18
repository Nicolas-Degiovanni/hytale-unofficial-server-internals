---
description: Architectural reference for SensorEntityPrioritiserDefault
---

# SensorEntityPrioritiserDefault

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.prioritisers
**Type:** Strategy / Utility

## Definition
```java
// Signature
public class SensorEntityPrioritiserDefault implements ISensorEntityPrioritiser {
```

## Architecture & Concepts
The SensorEntityPrioritiserDefault serves as a baseline implementation of the ISensorEntityPrioritiser interface. Its primary architectural role is to act as a simple, deterministic tie-breaker when an NPC's sensor system must choose between two potential targets: one player and one NPC.

The core logic is fundamentally a proximity check. It calculates the squared distance from a reference point (typically the sensing NPC's position) to both the player and the other NPC, selecting the closer of the two.

A key design choice is its delegation of the distance calculation to the NPC's active MotionController. This decouples the prioritiser from the specifics of movement and pathfinding, allowing it to correctly factor in navigation constraints like whether to use a 2D projected distance or a full 3D Euclidean distance.

This class also provides instances of the inner DefaultPrioritiser class, which is a rudimentary implementation of IEntityByPriorityFilter. This filter is extremely basic, latching onto the first entity it evaluates and retaining it as the highest priority. The main class, however, reports that it does not provide filters via the providesFilters method, making this inner class's role secondary to the primary pickTarget functionality.

## Lifecycle & Ownership
- **Creation:** An instance of SensorEntityPrioritiserDefault is a standard Java object. It is typically instantiated by a higher-level NPC component, such as a Sensor or a Behavior, during its own initialization. It is not managed by a dependency injection framework and is not a singleton.
- **Scope:** The lifetime of a SensorEntityPrioritiserDefault instance is tightly coupled to its owner. It persists as long as the parent NPC component that created it exists.
- **Destruction:** The object is eligible for garbage collection once its owning component is destroyed and all references to it are released. It does not manage any native resources and has no explicit cleanup or destruction method.

## Internal State & Concurrency
- **State:** The primary SensorEntityPrioritiserDefault class is stateless and immutable after construction. Its fields are final references to its inner prioritiser objects. However, the nested DefaultPrioritiser class is **stateful**. It contains a mutable *target* field which caches the first entity it processes. This state must be manually reset between sensing cycles by calling its cleanup method.

- **Thread Safety:** The main class is inherently thread-safe due to its stateless nature. The core pickTarget method is a pure function of its inputs.
    **WARNING:** The nested DefaultPrioritiser class is **not thread-safe**. Its test method performs a check-then-set operation on the *target* field, which is a classic race condition. It is designed to be used exclusively within a single-threaded context, such as the main server tick loop for a specific NPC. Unsynchronized access from multiple threads will lead to unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getNPCPrioritiser() | IEntityByPriorityFilter | O(1) | Returns the pre-allocated filter for NPC entities. |
| getPlayerPrioritiser() | IEntityByPriorityFilter | O(1) | Returns the pre-allocated filter for Player entities. |
| pickTarget(...) | Ref<EntityStore> | O(1) | The core decision-making method. Selects the closer of two entity references. Asserts that transform components exist. |
| providesFilters() | boolean | O(1) | Always returns false. Indicates this prioritiser does not contribute to the main filter list. |
| buildProvidedFilters(...) | void | O(1) | No-op. Does not add any filters to the provided list. |

## Integration Patterns

### Standard Usage
This class is intended to be used by an NPC's sensor system to resolve targeting conflicts. The system provides the two candidates, and this class returns the definitive target.

```java
// Within an NPC's sensor update logic
ISensorEntityPrioritiser prioritiser = new SensorEntityPrioritiserDefault();

// playerRef and npcRef are potential targets found by the sensor
Ref<EntityStore> chosenTarget = prioritiser.pickTarget(
    sensingNpcRef,
    sensingNpcRole,
    sensingNpcPosition,
    playerRef,
    npcRef,
    false, // Use 3D distance
    entityComponentStore
);

// The NPC behavior system now acts upon chosenTarget
```

### Anti-Patterns (Do NOT do this)
- **Misinterpreting its Purpose:** Do not use this class expecting sophisticated logic. It is a simple distance check and nothing more. It does not account for threat, line of sight, or behavioral state.
- **State Mismanagement:** Do not reuse the inner DefaultPrioritiser instances across multiple independent sensor ticks without calling the cleanup method. Failure to do so will cause the prioritiser to be "stuck" on a previously cached target.
- **Concurrent Access:** Never share an instance of the inner DefaultPrioritiser across multiple threads. Its state is not protected by locks and is unsafe for concurrent modification.

## Data Pipeline
This component acts as a selection node within a larger data flow, not as a processing pipeline itself. It takes two entity data streams and selects one to pass forward.

> Flow:
> NPC Sensor System Scans Area -> Identifies Potential Player and NPC Targets -> **SensorEntityPrioritiserDefault.pickTarget** -> Returns Single Winning Entity Reference -> NPC Behavior Tree or Targeting System

