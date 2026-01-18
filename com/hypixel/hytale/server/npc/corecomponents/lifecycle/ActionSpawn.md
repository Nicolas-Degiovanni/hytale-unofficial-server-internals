---
description: Architectural reference for ActionSpawn
---

# ActionSpawn

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle
**Type:** Transient State Object

## Definition
```java
// Signature
public class ActionSpawn extends ActionBase {
```

## Architecture & Concepts
ActionSpawn is a concrete implementation of the `ActionBase` contract, forming a fundamental component of the server-side NPC Behavior System. It encapsulates the complex, multi-stage process of an entity spawning one or more other entities into the world.

This class acts as a high-level orchestrator, mediating between several critical server systems:
*   **NPC Behavior System:** As an Action, it is invoked by an NPC's `Role` (its "brain") when a specific state or behavior tree node is activated.
*   **Spawning System:** It uses `SpawningContext` to perform environmental validation, ensuring that new entities are placed in valid, non-obstructed locations.
*   **Entity Component System (ECS):** It directly requests the creation of new entities and components via the `NPCPlugin` and manipulates component data on the new entity through the `Store<EntityStore>`.
*   **Flock System:** It contains logic to optionally create or join flocks via the `FlockPlugin`, enabling group behaviors for the spawned entities.
*   **Physics System:** It performs sophisticated physics calculations to determine initial velocity vectors, especially for its "launch at target" mode, which can compute projectile arcs.

The design of ActionSpawn decouples the declarative *intent* to spawn, defined in NPC asset files, from the imperative and stateful *execution* of the spawning logic. This allows for complex, dynamic spawning behaviors without cluttering the core NPC state machine.

## Lifecycle & Ownership
- **Creation:** ActionSpawn instances are never created directly with the `new` keyword. They are instantiated by the asset loading pipeline, specifically by a `BuilderActionSpawn` which parses configuration from an NPC's definition file. Each instance represents a single, configured spawn event.
- **Scope:** The object's lifetime is tied to the execution of a single spawn sequence. It is held as transient state within an NPC's `Role` component. The action may persist across multiple server ticks if it is configured with spawn delays, achieved by registering its `deferredSpawning` method as a callback with the `Role`.
- **Destruction:** Once the action completes (either by successfully spawning all entities or by exhausting its retry attempts), the `Role` releases its reference to the ActionSpawn instance. It is then eligible for garbage collection. An instance is **not** reusable.

## Internal State & Concurrency
- **State:** ActionSpawn is a highly stateful class. It maintains mutable internal state to track the progress of a spawn sequence, including counters (`spawnsLeft`, `maxTries`), timers (`spawnDelay`), and calculated spatial parameters (`yaw0`, `yawIncrement`). It also holds a `Ref` to the parent entity that initiated the action.

- **Thread Safety:** **This class is not thread-safe and must not be accessed from any thread other than the main server thread.** It is designed to operate exclusively within the single-threaded server update loop. All of its interactions with the world, the ECS `Store`, and other plugins assume non-concurrent access. Any off-thread modification will result in state corruption, race conditions, and server instability.

## API Surface
The primary contract is defined by its parent, `ActionBase`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Precondition check. Verifies that the action can begin. Fails if the target NPC type is invalid or if a required target is missing. |
| execute(...) | boolean | O(N) | Initiates the spawn sequence. Returns true upon completion, false if the action must continue on subsequent ticks. Complexity is proportional to the number of initial spawn attempts. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly in code. Its behavior is defined declaratively in an NPC asset file. The NPC's `Role` component, as part of its update tick, will select and execute the configured `ActionSpawn`.

The conceptual flow within the `Role` would be:

```java
// This is a conceptual example of how the NPC's Role would use the action.
// Do NOT instantiate ActionSpawn directly.

if (currentAction instanceof ActionSpawn spawnAction) {
    if (spawnAction.canExecute(selfRef, this, sensorInfo, dt, store)) {
        boolean isFinished = spawnAction.execute(selfRef, this, sensorInfo, dt, store);
        if (isFinished) {
            // Transition to the next state/action
            currentAction = null;
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** It is forbidden to use `new ActionSpawn()`. The object is complex to configure and must be constructed via the asset building pipeline to ensure all parameters are correctly initialized.
- **State Reuse:** An ActionSpawn instance is a single-use object. Do not attempt to "reset" its state and re-execute it. A new action must be fetched from the NPC's configuration for a subsequent spawn event.
- **Off-Thread Execution:** Calling `execute` or any other method from a worker thread is strictly prohibited and will lead to critical server errors.

## Data Pipeline
The flow of data and control for a spawn event is a multi-system process orchestrated by ActionSpawn.

> Flow:
> NPC Asset File -> `BuilderActionSpawn` -> **ActionSpawn Instance** -> `Role.execute()` -> `SpawningContext.canSpawn()` -> `NPCPlugin.spawnEntity()` -> New Entity in `EntityStore` -> **ActionSpawn.postSpawn()** -> `FlockPlugin.join()` / `Role.forceVelocity()` -> `NewSpawnStartTickingSystem`

