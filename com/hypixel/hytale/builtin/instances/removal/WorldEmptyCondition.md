---
description: Architectural reference for WorldEmptyCondition
---

# WorldEmptyCondition

**Package:** com.hypixel.hytale.builtin.instances.removal
**Type:** Utility / Strategy

## Definition
```java
// Signature
public class WorldEmptyCondition implements RemovalCondition {
```

## Architecture & Concepts
The WorldEmptyCondition is a specific implementation of the RemovalCondition strategy interface. Its primary role within the server architecture is to provide the logic for determining when a game world instance is considered "empty" and can be safely shut down to conserve system resources. This is a critical component of the server's dynamic world lifecycle management.

The condition is more sophisticated than a simple player count check. It embodies a state machine with three key states:
1.  **Awaiting First Player:** A newly created world starts in this state. A configurable timeout (`timeoutSeconds`) begins, acting as a safety net. If no player joins before the timeout expires, the world is marked for removal. This prevents orphaned worlds created by players who disconnect during the initial connection process.
2.  **Player Present:** Once a player joins, the condition transitions to this state. The safety timeout is cleared, and the world is considered active and will not be removed.
3.  **Empty After Presence:** If all players leave the world *after* at least one player was present, the condition is immediately met, and the world is marked for removal on the next check.

This logic ensures that worlds are not prematurely destroyed while being provisioned, but are aggressively reclaimed once they have served their purpose and become vacant.

### Lifecycle & Ownership
-   **Creation:** Instances are typically not created directly in code. They are deserialized from server configuration files via the provided `CODEC`. The server's world management system instantiates this class when loading a world template that specifies this removal condition. A default, shared instance is also available via the static `INSTANCE` field.
-   **Scope:** The lifetime of a WorldEmptyCondition object is tied to the world template or configuration that defines it. It is a lightweight, stateless object that persists as long as its configuration is loaded.
-   **Destruction:** The object holds no native resources and is managed by the Java garbage collector. It is reclaimed when the server configuration that references it is unloaded.

## Internal State & Concurrency
-   **State:** A WorldEmptyCondition instance is effectively immutable. Its only state, `timeoutSeconds`, is configured at creation and is not modified during its lifetime. The operational state it evaluates (player counts, timers) is stored externally in world-specific resources like `InstanceDataResource` and `TimeResource`. This separation makes the condition itself a reusable, stateless strategy.
-   **Thread Safety:** The class is inherently thread-safe due to its immutable nature. However, the `shouldRemoveWorld` method operates on external, mutable state.

    **WARNING:** The caller, typically the world management service, is responsible for ensuring that `shouldRemoveWorld` is invoked from a thread that has exclusive or synchronized access to the world's data store. Invoking this method from an unsynchronized, concurrent context can lead to severe race conditions regarding player join/leave events and timer updates. It is designed to be called from the main server tick thread for the corresponding world.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shouldRemoveWorld(Store store) | boolean | O(1) | Evaluates the world's state against the emptiness criteria. Returns true if the world should be removed. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in typical gameplay logic. It is a system-level component configured declaratively. The server's world instance manager consumes it as part of its periodic lifecycle checks.

A conceptual example of how the *system* uses it:
```java
// Executed by the WorldInstanceManager during its update tick
for (ManagedWorld world : activeWorlds) {
    RemovalCondition condition = world.getRemovalCondition(); // Returns an instance of WorldEmptyCondition
    if (condition.shouldRemoveWorld(world.getStore())) {
        scheduleWorldForShutdown(world);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Avoid using `new WorldEmptyCondition()` unless for specific, isolated testing. The standard pattern is to use the static `INSTANCE` or allow the server to create it via its configuration `CODEC`.
-   **Asynchronous Evaluation:** Do not call `shouldRemoveWorld` from an arbitrary worker thread. The method reads from multiple resources (`InstanceDataResource`, `World`, `TimeResource`) that are not guaranteed to be thread-safe and are mutated by the main game loop. This will lead to inconsistent reads and incorrect lifecycle decisions.

## Data Pipeline
The evaluation of this condition is the final step in a data flow triggered by server state changes, primarily player connection events and the passage of time.

> Flow:
> Player Disconnect Event -> World Player Count updated to 0 -> Server Tick -> World Instance Manager iterates -> **WorldEmptyCondition.shouldRemoveWorld()** reads world state -> Returns `true` -> World Manager schedules world for removal.
>
> Alternate Flow (Timeout):
> World Created -> **WorldEmptyCondition** logic sets timeout in `InstanceDataResource` -> Server Tick -> Time passes -> `TimeResource` advances -> **WorldEmptyCondition.shouldRemoveWorld()** checks timer -> Returns `true` -> World Manager schedules world for removal.

