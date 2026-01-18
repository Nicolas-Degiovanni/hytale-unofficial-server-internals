---
description: Architectural reference for VoidInvasionPortalsSpawnSystem
---

# VoidInvasionPortalsSpawnSystem

**Package:** com.hypixel.hytale.builtin.portals.systems.voidevent
**Type:** System (ECS)

## Definition
```java
// Signature
public class VoidInvasionPortalsSpawnSystem extends DelayedEntitySystem<EntityStore> {
```

## Architecture & Concepts

The VoidInvasionPortalsSpawnSystem is a server-side, time-delayed process within the Hytale Entity Component System (ECS) framework. Its sole responsibility is to periodically spawn new "invasion portals" associated with an active VoidEvent in the world.

This system operates on a fixed 2.0-second interval, as defined by its superclass constructor. It queries the world for entities that possess the VoidEvent component, which acts as the central state-tracking entity for a world-level invasion event.

The core architectural pattern is an **asynchronous state machine** managed across multiple ticks. To avoid blocking the main server thread with expensive world-generation queries, the system initiates a non-blocking search for a valid portal location. This search is encapsulated in a CompletableFuture. The system's internal state, represented by this future, determines its behavior in subsequent ticks:

1.  **Idle State:** No search is active. The system checks if more portals are needed. If so, it transitions to the Searching State.
2.  **Searching State:** An asynchronous search for a spawn position is in progress. The system does nothing but wait for the future to complete.
3.  **Resolution State:** The future has completed. The system retrieves the result, creates the new portal entity and associated block via the CommandBuffer, and transitions back to the Idle State.

This design ensures that server performance remains stable, as the computationally intensive task of finding a valid, non-overlapping position near a player is offloaded from the main game loop.

## Lifecycle & Ownership

-   **Creation:** Instantiated automatically by the server's ECS System Manager upon world initialization. Developers do not and should not create instances of this class manually.
-   **Scope:** The system's lifecycle is bound to the server-side World it operates within. A single instance exists for the duration of a running world session.
-   **Destruction:** The instance is marked for garbage collection when the associated World is unloaded or the server shuts down.

## Internal State & Concurrency

-   **State:** This system is stateful across ticks. It contains a single mutable field, `findPortalSpawnPos` of type CompletableFuture. This field is `null` when the system is idle and holds a future object when an asynchronous position search is active. This state dictates the system's logic flow within its `tick` method.

-   **Thread Safety:** This class is **not thread-safe** and must only be accessed by the single thread that manages the ECS update loop. All interactions with the world state are mediated through the CommandBuffer, which guarantees that entity and component modifications are queued and executed safely at the end of the tick. The asynchronous operation launched via `CompletableFuture.supplyAsync` is explicitly executed on the World's dedicated thread pool, with the result being safely consumed back on the main thread in a subsequent tick.

## API Surface

The public API is exclusively for consumption by the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(1) | The primary entry point, called by the ECS scheduler. Executes one step of the state machine. Its own complexity is constant, but it may trigger an asynchronous operation of high complexity. |
| getQuery() | Query | O(1) | Defines the component signature for entities this system will process. Returns a query for entities with the VoidEvent component. |

## Integration Patterns

### Standard Usage

This system functions automatically. A developer or gameplay script triggers its behavior by adding a VoidEvent component to an entity within the world. The system will then detect this entity and begin its portal spawning logic.

```java
// To activate the system, a gameplay script or other system
// would create an entity with the VoidEvent component.

// (Conceptual Example)
Holder<EntityStore> eventHolder = EntityStore.REGISTRY.newHolder();
eventHolder.addComponent(VoidEvent.getComponentType(), new VoidEvent());
commandBuffer.addEntity(eventHolder, AddReason.SCRIPT);

// The VoidInvasionPortalsSpawnSystem will now automatically
// begin processing this entity on its next scheduled tick.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new VoidInvasionPortalsSpawnSystem()`. The ECS framework is responsible for its creation and lifecycle management.
-   **Manual Ticking:** Never call the `tick` method directly. Doing so will bypass the engine's scheduler, break the CommandBuffer pattern, and introduce severe race conditions.
-   **External State Modification:** Do not access or modify the internal `findPortalSpawnPos` future from another class. This will corrupt the system's state machine.

## Data Pipeline

The system's operational flow for spawning a single portal spans multiple ticks.

> Flow:
> **Tick 1 (Initiation):**
> ECS Scheduler invokes `tick` -> System checks `findPortalSpawnPos` (is null) -> System verifies portal count is below MAX_PORTALS -> A random player's position is located -> An asynchronous `SearchCone` query is launched on a worker thread -> The resulting `CompletableFuture` is stored in `findPortalSpawnPos`.
>
> **Tick N (Waiting):**
> ECS Scheduler invokes `tick` -> System checks `findPortalSpawnPos` (is not null) -> `isDone()` returns false -> System does nothing and waits.
>
> **Tick N+1 (Resolution):**
> ECS Scheduler invokes `tick` -> System checks `findPortalSpawnPos` (is not null) -> `isDone()` returns true -> The result (a Vector3d position) is retrieved -> A `CommandBuffer.addEntity` command is queued to create a new VoidSpawner entity -> An asynchronous `world.getChunkAsync` task is scheduled to place the physical portal block -> `findPortalSpawnPos` is reset to `null`.

