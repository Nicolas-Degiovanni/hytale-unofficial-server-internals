---
description: Architectural reference for PortalRemovalCondition
---

# PortalRemovalCondition

**Package:** com.hypixel.hytale.builtin.portals.integrations
**Type:** Transient

## Definition
```java
// Signature
public class PortalRemovalCondition implements RemovalCondition {
```

## Architecture & Concepts
The PortalRemovalCondition is a specialized implementation of the RemovalCondition strategy interface. Its primary function is to define the policy for when a temporary "portal world" should be automatically unloaded and removed by the server's instance management system.

This class operates as a composite condition, combining two distinct removal policies:
1.  **Timeout-based:** A world is removed after a configurable duration has passed since the first player joined. This is managed by the internal TimeoutCondition.
2.  **Emptiness-based:** A world is removed if it remains empty (contains no players) for a fixed duration. This is managed by the internal WorldEmptyCondition.

The core logic prioritizes the timeout *after* a player has entered the world. If a player has joined the instance, the timeout condition is the primary trigger for removal. The emptiness condition acts as a fallback, ensuring that even unvisited or quickly abandoned worlds are eventually cleaned up.

Crucially, this component is a stateless evaluator. The actual state, such as the timeout expiration instant and the flag indicating player presence, is not stored within this object. Instead, it is persisted within the world's data stores, specifically in the InstanceDataResource. This design decouples the removal logic from the world's state, allowing the condition to be a lightweight, reusable component.

### Lifecycle & Ownership
-   **Creation:** PortalRemovalCondition instances are primarily intended to be created via deserialization from world configuration files using the static CODEC. This allows world designers to define removal policies declaratively. Direct instantiation via its constructor is possible but typically reserved for testing or programmatic world generation.
-   **Scope:** An instance of this class is scoped to a single world definition. It is loaded when the world is initialized and persists for the lifetime of that world instance.
-   **Destruction:** The object is eligible for garbage collection once the world it is associated with is fully unloaded and all references to its configuration are released.

## Internal State & Concurrency
-   **State:** This class is a stateful container for its configuration (e.g., timeout duration) but is a stateless processor regarding world data. All operational state, such as the calculated timeout instant, is read from and written to the World's ChunkStore and EntityStore via shared Resources like InstanceDataResource and TimeResource.
-   **Thread Safety:** This class is not internally thread-safe and contains no synchronization primitives. It is designed to be invoked by the server's main world update thread or a dedicated instance management thread that has exclusive or synchronized access to the World object and its underlying data stores.

    **Warning:** Concurrent invocations of its methods on the same World object from multiple threads will lead to race conditions and unpredictable behavior. All access must be externally synchronized.

## API Surface
The public API provides methods to inspect and control the removal timer associated with a specific world.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shouldRemoveWorld(Store) | boolean | O(1) | The core evaluation method. Returns true if the world meets the criteria for removal. |
| getElapsedSeconds(World) | double | O(1) | Returns the number of seconds that have passed since the timeout timer was started. |
| getRemainingSeconds(World) | double | O(1) | Returns the number of seconds left before the timeout condition is met. |
| setRemainingSeconds(World, double) | void | O(1) | Overwrites the world's timeout, setting a new remaining duration. |

## Integration Patterns

### Standard Usage
This component is not intended for direct use in typical gameplay logic. It is consumed by a higher-level server system responsible for managing the lifecycle of game instances.

```java
// Conceptual usage by an InstanceManagementSystem

for (World world : activePortalWorlds) {
    RemovalCondition condition = world.getRemovalCondition(); // Returns a PortalRemovalCondition

    // The system checks the condition on each tick or at a fixed interval
    if (condition.shouldRemoveWorld(world.getChunkStore().getStore())) {
        scheduleForUnload(world);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Do not call shouldRemoveWorld from gameplay scripts. World removal is a core server process and should be managed centrally.
-   **State Mismanagement:** Do not use setRemainingSeconds to create long-lived worlds. This component is designed for *temporary* instances. Abusing the timer can interfere with the server's ability to manage resources.
-   **Incorrect Configuration:** Applying this condition to a persistent, non-portal world is an error. The logic is tightly coupled to the lifecycle of a temporary instance that expects to be cleaned up.

## Data Pipeline
PortalRemovalCondition participates in a control flow rather than a traditional data pipeline. It acts as a predicate in the server's instance management loop.

> **Control Flow:**
> Instance Manager Tick -> Get World's **PortalRemovalCondition** -> `shouldRemoveWorld()` -> Reads `InstanceDataResource` & `TimeResource` from World Stores -> Returns `true` -> Instance Manager initiates world unload sequence.

