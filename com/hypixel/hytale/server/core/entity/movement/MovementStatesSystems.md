---
description: Architectural reference for MovementStatesSystems
---

# MovementStatesSystems

**Package:** com.hypixel.hytale.server.core.entity.movement
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class MovementStatesSystems {
    public static class AddSystem extends HolderSystem<EntityStore> { /* ... */ }
    public static class PlayerInitSystem extends RefSystem<EntityStore> { /* ... */ }
    public static class TickingSystem extends EntityTickingSystem<EntityStore> { /* ... */ }
}
```

## Architecture & Concepts

The MovementStatesSystems class is not a singular object but a static container for a group of related systems within the server's Entity Component System (ECS) framework. These systems collectively manage the lifecycle, persistence, and network synchronization of an entity's movement state, represented by the MovementStatesComponent.

Their responsibilities are partitioned into three distinct phases, each handled by a dedicated inner class:

*   **AddSystem:** This system acts as an initializer. It subscribes to the creation of any living entity via the AllLegacyLivingEntityTypesQuery. Upon entity creation, its sole responsibility is to attach a default, empty MovementStatesComponent, ensuring that every relevant entity has the necessary data structure for movement tracking from its inception.

*   **PlayerInitSystem:** This system handles state restoration specifically for player entities. When a player entity is added to the world, this system retrieves the player's last known movement state from persistent storage (PlayerWorldData). It then applies this saved state to the newly created MovementStatesComponent. This is critical for continuity, ensuring a player who logs out while flying or crouching logs back in with the same state.

*   **TickingSystem:** This is the high-frequency, performance-critical synchronization engine. It runs every server tick and performs a dirty check on the MovementStatesComponent for all visible entities. If an entity's movement state has changed since the last tick, this system serializes the new state into a ComponentUpdate packet and broadcasts it to all clients that are currently tracking that entity. This is the core mechanism that allows players to see each other's animations and movement changes in real-time.

## Lifecycle & Ownership

-   **Creation:** The inner system classes are not designed for manual instantiation. They are discovered via reflection and instantiated by the server's central ECS System Registry during the server bootstrap sequence. The framework is responsible for injecting their dependencies, such as ComponentType handles.
-   **Scope:** A single instance of each inner system persists for the entire lifetime of the server process. They are designed as stateless services that operate on streams of component data managed by the ECS framework.
-   **Destruction:** The systems are destroyed during an orderly server shutdown when the ECS System Registry is torn down.

## Internal State & Concurrency

-   **State:** The MovementStatesSystems container is stateless. The inner systems are also fundamentally stateless; they hold immutable references to component types but contain no per-entity or per-session instance data. All relevant state is stored externally in components within the EntityStore.

-   **Thread Safety:** The TickingSystem is explicitly designed for parallel execution, as declared by its `isParallel` method. The Hytale ECS scheduler can safely execute its `tick` method for different batches of entities (ArchetypeChunks) across multiple worker threads. This parallelism is a critical performance feature.

    **WARNING:** This safety guarantee relies on the system's statelessness. All entity data modifications must be funneled through the provided Store and CommandBuffer parameters. Direct mutation of shared state from within these systems will break thread safety and lead to severe data corruption and race conditions.

## API Surface

The public API of these systems is their contract with the ECS framework, not with a typical developer. The framework invokes these methods at specific points in the entity lifecycle.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AddSystem.onEntityAdd | void | O(1) | Framework-invoked. Attaches a MovementStatesComponent to a new entity. |
| PlayerInitSystem.onEntityAdded | void | O(1) | Framework-invoked. Hydrates a player's MovementStatesComponent from persistent storage. |
| TickingSystem.tick | void | O(N) | Framework-invoked each tick. Checks for state changes and queues network packets for N entities. |

## Integration Patterns

### Standard Usage

Developers do not interact with these systems directly. Interaction is implicit and declarative, driven by the ECS architecture.

To trigger this logic, a developer would simply create an entity that satisfies the system's queries. For example, creating a living entity will automatically cause the AddSystem to run for it. Subsequently modifying its MovementStatesComponent will cause the TickingSystem to detect the change and synchronize it.

```java
// This is a conceptual example.
// The act of creating a player entity and adding it to the world
// is sufficient to trigger the entire MovementStatesSystems pipeline.

// 1. A new player entity is created by another system.
Ref<EntityStore> playerEntity = world.createPlayerEntity();

// 2. AddSystem automatically runs, adding a MovementStatesComponent.

// 3. PlayerInitSystem automatically runs, loading saved states.

// 4. Later, a physics system might change the state.
MovementStatesComponent states = store.getComponent(playerEntity, MOVEMENT_STATES_TYPE);
states.getMovementStates().jumping = true;

// 5. On the next tick, TickingSystem will detect this change and broadcast it.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create instances of these systems using `new`. The ECS framework manages their lifecycle and dependency injection. Manual creation will result in a non-functional, disconnected object.
-   **Manual Invocation:** Do not call methods like `tick` or `onEntityAdd` directly. This bypasses the ECS scheduler, threading model, and command buffer, which will corrupt game state and cause unpredictable crashes.
-   **Stateful Systems:** Avoid adding mutable instance fields to these system classes to store temporary data. State must be stored in components to ensure correctness, especially in a parallel execution environment.

## Data Pipeline

The flow of data through these systems is event-driven and sequential, based on the entity lifecycle.

**Pipeline 1: New Player Entity Initialization**
> Flow:
> Player Joins Server -> Entity is created -> **AddSystem.onEntityAdd** (Attaches default component) -> **PlayerInitSystem.onEntityAdded** (Reads PlayerWorldData, populates component from storage) -> Entity is fully initialized in the world.

**Pipeline 2: Real-time State Synchronization**
> Flow:
> Game Logic (e.g., Physics) modifies MovementStatesComponent -> Server Tick Begins -> **TickingSystem.tick** (Compares current vs. last-sent state) -> Change Detected -> `queueUpdatesFor` is called -> ComponentUpdate packet is queued for all nearby clients -> Network Layer sends packet.

