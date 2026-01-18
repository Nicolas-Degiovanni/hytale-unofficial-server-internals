---
description: Architectural reference for LocalSpawnControllerSystem
---

# LocalSpawnControllerSystem

**Package:** com.hypixel.hytale.server.spawning.local
**Type:** System (ECS)

## Definition
```java
// Signature
public class LocalSpawnControllerSystem extends TickingSystem<EntityStore> {
```

## Architecture & Concepts

The LocalSpawnControllerSystem is a server-side ECS TickingSystem responsible for initiating the entity spawning process in the vicinity of a controlling entity, typically a player. It acts as the primary decision-making engine for *when* and *where* to place spawn beacons, which are precursors to actual entity spawning.

This system does not spawn creatures directly. Instead, its sole output is the creation of **LegacySpawnBeaconEntity** entities. This decouples the complex logic of evaluating spawn conditions from the simpler mechanics of entity instantiation. Another system is expected to consume these beacons and perform the final spawn.

Architecturally, this system operates on a fixed-frequency timer, defined by **RUN_FREQUENCY_SECONDS**, rather than every game tick. This is a critical performance-saving measure, as the logic it executes is computationally expensive. For each eligible controlling entity (e.g., a player), the system performs a multi-stage evaluation:

1.  **Environmental Query:** It assesses the current environment of the controller, including weather and time of day, to determine a set of potential spawns.
2.  **Spatial Cooldown:** It queries a **SpatialResource** to find existing spawn beacons in the area. This prevents over-population and duplicate beacon placement for the same spawn type.
3.  **Condition Validation:** It performs fine-grained checks, most notably a complex light-level analysis of the surrounding terrain.
4.  **Beacon Placement:** If all conditions are met, it creates a new beacon entity in the world's EntityStore.

This system is central to the dynamic and environmentally-aware spawning behavior of the Hytale server.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's central System Registry during world initialization. It is not intended for manual creation.
-   **Scope:** Singleton per world instance. It persists for the entire lifecycle of a running game world.
-   **Destruction:** Decommissioned and garbage collected when the world is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** The system class itself is stateless, with its dependencies injected via the constructor and stored in final fields. However, it operates on highly mutable, shared state within the **EntityStore** and global resources like **LocalSpawnState**. It uses the LocalSpawnState resource as a temporary, per-tick buffer to batch-process controllers.

-   **Thread Safety:** **This system is not thread-safe.** It is designed to be executed exclusively by the main server world tick thread. The use of thread-local collections (**SpatialResource.getThreadLocalReferenceList**) and direct mutation of the EntityStore mandate single-threaded access.

    **WARNING:** Calling the tick method from any thread other than the world's primary update thread will lead to data corruption, race conditions, and server instability.

## API Surface

The public API is exclusively for engine-level integration. Game logic developers should not interact with this API directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | O(P * B * D) | The main entry point called by the ECS runner. P is the number of active players, B is the number of possible beacon types for an environment, and D is the density of entities in the spatial query. **Do not call directly.** |

## Integration Patterns

### Standard Usage
Developers do not call this system. To make an entity (such as a player) a focal point for local spawning, add the **LocalSpawnController** component to it. The system will automatically discover and process the entity on its next scheduled run.

```java
// Example: Making a player entity a spawn controller
// This code would exist within a system that manages player creation.

Ref<EntityStore> playerEntity = ...;
store.ensureComponent(playerEntity, LocalSpawnController.class);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new LocalSpawnControllerSystem()`. The system is managed by the engine's dependency injection and lifecycle framework. Manual creation will result in a non-functional system that is not registered to receive ticks.
-   **Manual Ticking:** Do not call the `tick` method manually. This bypasses the rate-limiting timer and can cause severe server performance degradation. It also breaks assumptions about the execution order of systems.
-   **External State Mutation:** Do not modify the `LocalSpawnState` resource while this system is potentially running. It is used internally for batching and is cleared at the end of the tick.

## Data Pipeline
The system transforms environmental and player state into potential spawn locations, represented as beacon entities.

> Flow:
> Player Position & World State -> **LocalSpawnControllerSystem (Tick)** -> Environment & Light Level Analysis -> Spatial Query for Existing Beacons -> **LegacySpawnBeaconEntity Creation** -> EntityStore Update -> Downstream Spawning System

---

