---
description: Architectural reference for SensorEntityPrioritiserAttitude
---

# SensorEntityPrioritiserAttitude

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.prioritisers
**Type:** Strategy Object

## Definition
```java
// Signature
public class SensorEntityPrioritiserAttitude implements ISensorEntityPrioritiser {
```

## Architecture & Concepts
The SensorEntityPrioritiserAttitude is a core component of the NPC Artificial Intelligence engine, specifically operating within the target selection subsystem. It provides a concrete strategy for an NPC's "sensor" to determine the most important entity to engage with from a group of potential targets.

This class implements the logic for ranking entities based on their **Attitude** relative to the NPC (e.g., Hostile, Friendly, Neutral). An NPC's behavior asset defines an ordered list of attitudes, such as `[HOSTILE, NEUTRAL]`. This class consumes that configuration to ensure the NPC will always prioritize a hostile entity over a neutral one, regardless of distance or other factors.

It serves two primary functions:
1.  **Target Prioritization:** It provides stateful filter objects (via getNPCPrioritiser) that can be used to iterate over a collection of entities and find the single highest-priority target according to the configured attitude list.
2.  **Pre-filtering:** It provides a stateless `EntityFilterAttitude` to optimize the initial entity detection phase, allowing the sensor system to immediately discard entities with attitudes not present in the priority list.

This component is fundamental to creating differentiated NPC behaviors, allowing designers to specify whether an NPC is aggressive, defensive, or indifferent to its surroundings.

### Lifecycle & Ownership
-   **Creation:** This object is instantiated exclusively by the server's asset loading pipeline. It is constructed via its corresponding builder, `BuilderSensorEntityPrioritiserAttitude`, when an NPC's behavior profile is loaded from configuration files. It is not intended for runtime instantiation.
-   **Scope:** An instance of this class persists for the lifetime of the NPC's `Role` definition. It is effectively a static, configured component of an NPC's "brain".
-   **Destruction:** The object is marked for garbage collection when the server unloads the associated NPC assets or behavior profiles. There is no explicit destruction method. Note that the inner `AttitudePrioritiser` class has its own distinct, shorter lifecycle managed by the sensor system.

## Internal State & Concurrency
-   **State:** The primary state of this class is the `attitudeByPriority` array, which is `final` and set only during construction. Consequently, the SensorEntityPrioritiserAttitude object itself is **immutable** and can be considered a stateless configuration holder after initialization.

    **WARNING:** The inner class `AttitudePrioritiser` is highly stateful and **mutable**. It maintains a `targetsByAttitude` array that is populated during a filtering operation. This design makes it single-use per operation.

-   **Thread Safety:** The outer SensorEntityPrioritiserAttitude class is **thread-safe** due to its immutability. Its methods can be safely invoked from any thread.

    However, the `AttitudePrioritiser` object returned by `getNPCPrioritiser` and `getPlayerPrioritiser` is **NOT thread-safe**. It is designed for a strict, single-threaded lifecycle of `init()`, followed by a series of `test()` calls, a single `getHighestPriorityTarget()` call, and finally `cleanup()`. Concurrent access will lead to race conditions and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerWithSupport(Role) | void | O(1) | Registers requirements with the world system. **CRITICAL:** Must be called before use to ensure the Attitude Cache is active. |
| getNPCPrioritiser() | IEntityByPriorityFilter | O(1) | Returns a new, stateful prioritiser instance for evaluating NPC targets. |
| getPlayerPrioritiser() | IEntityByPriorityFilter | O(1) | Returns a new, stateful prioritiser instance for evaluating Player targets. |
| pickTarget(...) | Ref<EntityStore> | O(N) | A specialized utility to select between exactly two targets (one player, one NPC). If priorities are equal, it defaults to distance. |
| providesFilters() | boolean | O(1) | Returns true, indicating this component can supply an optimization filter. |
| buildProvidedFilters(List) | void | O(1) | Populates a list with its associated `EntityFilterAttitude` for pre-filtering. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers. It is configured via NPC asset files and managed by the NPC sensor and behavior systems. The conceptual flow within the engine is as follows.

```java
// Engine-level code within an NPC sensor update
// 1. Obtain the configured prioritiser from the NPC's Role
ISensorEntityPrioritiser prioritiser = npc.getRole().getSensorPrioritiser();

// 2. Get a fresh, stateful filter for this update cycle
IEntityByPriorityFilter filter = prioritiser.getNPCPrioritiser();
filter.init(npc.getRole());

// 3. Iterate through potential targets
for (Ref<EntityStore> target : nearbyEntities) {
    // The test method populates the filter's internal state
    filter.test(npc.getRef(), target, componentAccessor);
}

// 4. Retrieve the single best target based on attitude
Ref<EntityStore> bestTarget = filter.getHighestPriorityTarget();

// 5. Clean up the stateful filter to prevent memory leaks
filter.cleanup();

// 6. Pass the bestTarget to the behavior system
if (bestTarget != null) {
    npc.getBehaviorTree().setTarget(bestTarget);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new SensorEntityPrioritiserAttitude()`. The object must be constructed through the asset pipeline to receive its priority configuration. Bypassing this will result in a non-functional component.
-   **Reusing Prioritiser Instances:** The `IEntityByPriorityFilter` returned by `getNPCPrioritiser` is stateful. It must not be shared across different sensor updates or different NPCs. A new instance must be retrieved for each distinct filtering operation, and its `cleanup` method must be called after use.

## Data Pipeline
The flow of entity data through this system is designed to efficiently select a single target from many potential candidates.

> Flow:
> List of Nearby Entities -> **EntityFilterAttitude** (Optional Pre-filter) -> Filtered List -> **AttitudePrioritiser.test()** (Iterative Evaluation) -> Populated Internal Target Cache -> **AttitudePrioritiser.getHighestPriorityTarget()** -> Single Highest Priority Entity -> NPC Behavior System

