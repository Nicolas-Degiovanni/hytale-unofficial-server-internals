---
description: Architectural reference for FlockSystems
---

# FlockSystems

**Package:** com.hypixel.hytale.server.flock
**Type:** Utility

## Definition
```java
// Signature
public class FlockSystems {
    // Contains static nested System classes
}
```

## Architecture & Concepts
The **FlockSystems** class is not a traditional object but a static container or namespace. It groups together several distinct but related ECS (Entity Component System) systems that collectively manage the lifecycle and behavior of the **Flock** and **FlockMembership** components.

This class acts as a manifest for the flocking feature's core logic, providing clear separation of concerns:
*   **Ticking:** Per-frame updates for active flocks.
*   **EntityRemoved:** Cleanup and dissolution logic when a flock is destroyed.
*   **PlayerChangeGameModeEventSystem:** Reaction to external game events that affect flock members.

By encapsulating these systems within a single outer class, the engine maintains a clean and discoverable API for the flocking module. Developers interact with the nested static classes, not the **FlockSystems** class itself.

---
description: Architectural reference for FlockSystems.EntityRemoved
---

# FlockSystems.EntityRemoved

**Package:** com.hypixel.hytale.server.flock
**Type:** Transient

## Definition
```java
// Signature
public static class EntityRemoved extends RefSystem<EntityStore> {
```

## Architecture & Concepts
**EntityRemoved** is a reactive ECS system responsible for maintaining data integrity when a flock's central entity is removed from the world. It acts as a finalizer or cleanup mechanism, preventing orphaned entities and inconsistent game states.

The system's core responsibility is to dissolve a flock when its anchor entity is removed. It queries for entities that possess a **Flock**, **UUIDComponent**, and **EntityGroup**. Upon detecting a removal, it iterates through all members of the flock's **EntityGroup** and systematically removes their **FlockMembership** component. This action effectively disbands the flock and returns the member entities to an independent state.

This system is critical for handling two distinct scenarios:
1.  **REMOVE:** A deliberate, in-game destruction of the flock. The system dissolves the group and marks associated transforms as dirty to ensure visual updates.
2.  **UNLOAD:** The chunk containing the flock entity is unloaded. The system performs a lighter-weight cleanup, preparing the flock for potential reloading without dissolving it permanently.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the **FlockPlugin** during server initialization. A new instance is created for each registered **Flock** component type.
-   **Scope:** The system's lifecycle is tied to the ECS scheduler. It persists as long as the **FlockPlugin** is active and registered with the world.
-   **Destruction:** Deregistered and garbage collected when the plugin is disabled or the server shuts down.

## Internal State & Concurrency
-   **State:** The system is stateful in its configuration, holding immutable references to the **ComponentType**s it operates on. During execution, it is stateless and its behavior is determined solely by the entity being processed.
-   **Thread Safety:** This system is designed to be managed by the Hytale ECS scheduler. The use of a **CommandBuffer** for all entity modifications is a critical concurrency control. All component removals are queued and executed at a safe synchronization point at the end of the tick, preventing race conditions with other systems that might be reading the same component data.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityRemove(ref, reason, store, cmd) | void | O(N) | Callback triggered by the ECS when a flock entity is removed. Complexity is O(N) where N is the number of members in the flock. |

## Integration Patterns

### Standard Usage
This system is not intended for direct invocation. It is registered with the ECS world by the plugin framework, and the engine automatically invokes its callbacks at the appropriate time.

```java
// Within the FlockPlugin initialization
ComponentType<EntityStore, MyFlock> flockType = ...;
RefSystem<EntityStore> removalSystem = new FlockSystems.EntityRemoved(flockType);

// The system is registered with the world's scheduler
world.getScheduler().addSystem(removalSystem);
```

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Never call **onEntityRemove** directly. The ECS scheduler is solely responsible for its execution based on entity lifecycle events. Manual calls will bypass the **CommandBuffer** and corrupt world state.
-   **Incorrect Registration:** Registering this system to run at the wrong stage of the game loop can lead to missed removals or state corruption.

## Data Pipeline
> Flow:
> Flock Entity Destroyed -> ECS Scheduler -> **EntityRemoved.onEntityRemove** -> CommandBuffer (Queues **FlockMembership** removal) -> End-of-Tick Sync -> World State Updated

---
description: Architectural reference for FlockSystems.PlayerChangeGameModeEventSystem
---

# FlockSystems.PlayerChangeGameModeEventSystem

**Package:** com.hypixel.hytale.server.flock
**Type:** Transient

## Definition
```java
// Signature
public static class PlayerChangeGameModeEventSystem extends EntityEventSystem<EntityStore, ChangeGameModeEvent> {
```

## Architecture & Concepts
This is a specialized event-handling system that decouples flock logic from player state management. Its sole purpose is to listen for **ChangeGameModeEvent** and ensure players are removed from any flocks if they switch to a non-participatory game mode, such as Creative or Spectator.

By reacting to events rather than polling player state every tick, this system provides a highly efficient and robust mechanism for maintaining logical consistency. It prevents scenarios where a player in an incompatible game mode could still be considered part of an active flock, which could lead to bugs in targeting, scoring, or other gameplay mechanics.

**WARNING:** The system's query is **Archetype.empty()**. This is a special case indicating that it does not iterate entities every tick. Instead, it is triggered exclusively by the event bus when a **ChangeGameModeEvent** is dispatched for any entity.

## Lifecycle & Ownership
-   **Creation:** Instantiated once by the **FlockPlugin** during server initialization.
-   **Scope:** Lives for the entire server session, registered as a listener on the ECS event bus.
-   **Destruction:** Deregistered and garbage collected on server shutdown.

## Internal State & Concurrency
-   **State:** This system is entirely stateless.
-   **Thread Safety:** Execution is managed by the ECS event bus. The **handle** method uses a **CommandBuffer** to queue the removal of the **FlockMembership** component, guaranteeing that the modification is applied in a thread-safe manner at the end of the current tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(index, chunk, store, cmd, event) | void | O(1) | Callback triggered by the ECS event bus when a **ChangeGameModeEvent** occurs. |

## Integration Patterns

### Standard Usage
This system is automatically managed by the **FlockPlugin**. No direct developer interaction is required after initial registration.

```java
// Within the FlockPlugin initialization
EntityEventSystem system = new FlockSystems.PlayerChangeGameModeEventSystem();

// The system is registered with the world's scheduler/event bus
world.getScheduler().addSystem(system);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation and Use:** Do not create instances of this class to manually process events. It must be registered with the engine's event system to function correctly.

## Data Pipeline
> Flow:
> Player GameMode Change -> **ChangeGameModeEvent** Fired -> ECS Event Bus -> **PlayerChangeGameModeEventSystem.handle** -> CommandBuffer (Queues **FlockMembership** removal) -> End-of-Tick Sync

---
description: Architectural reference for FlockSystems.Ticking
---

# FlockSystems.Ticking

**Package:** com.hypixel.hytale.server.flock
**Type:** Transient

## Definition
```java
// Signature
public static class Ticking extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
This is the primary "update" system for the **Flock** component. As an **EntityTickingSystem**, it is executed by the ECS scheduler once per game tick for every entity that has a **Flock** component.

Its main responsibility is to perform periodic, per-tick maintenance on the internal state of the **Flock** component. The observed implementation calls **flock.swapDamageDataBuffers()**, which strongly implies the use of a double-buffering pattern. This pattern is common in high-performance contexts to avoid read/write hazards within a single frame. One buffer accumulates incoming data (like damage events) during the tick, while the other, stable buffer is used for reads. At the end of the tick, this system swaps the buffers, making the newly collected data available for the next tick's calculations.

The system is designed for parallelism, as indicated by the **isParallel** method. The ECS scheduler can distribute the work of ticking all flock entities across multiple worker threads, making this a performance-critical component for servers with many active flocks.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the **FlockPlugin** during server initialization, configured with the specific **ComponentType** for the flock it will manage.
-   **Scope:** Persists for the server's lifetime, registered with the ECS scheduler.
-   **Destruction:** Deregistered and garbage collected on server shutdown.

## Internal State & Concurrency
-   **State:** The system holds an immutable reference to a **ComponentType**. It does not maintain any mutable state itself between ticks. All state is stored within the **Flock** components it operates on.
-   **Thread Safety:** The system is designed to be run in parallel. The ECS scheduler guarantees that the **tick** method will only be called for entities within a specific **ArchetypeChunk** on a single thread at a time. This data-local execution model, combined with the fact that it only modifies the component for the current entity *index*, ensures thread safety without explicit locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(1) | Executed once per tick for each entity with a **Flock** component. Performs internal state maintenance. |

## Integration Patterns

### Standard Usage
This system is registered by the plugin framework and should not be invoked directly.

```java
// Within the FlockPlugin initialization
ComponentType<EntityStore, MyFlock> flockType = ...;
EntityTickingSystem<EntityStore> tickingSystem = new FlockSystems.Ticking(flockType);

// The system is registered with the world's scheduler
world.getScheduler().addSystem(tickingSystem);
```

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Calling the **tick** method directly will disrupt the engine's scheduler and can lead to severe concurrency issues, such as processing the same component's state multiple times in one frame.

## Data Pipeline
> Flow:
> Game Tick Begins -> ECS Scheduler -> **Ticking.tick** (Invoked for each Flock entity, potentially in parallel) -> Mutates internal state of **Flock** component (e.g., swaps buffers) -> Game Tick Continues

