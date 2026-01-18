---
description: Architectural reference for NPCInteractionSystems
---

# NPCInteractionSystems

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Utility

## Architecture & Concepts
The NPCInteractionSystems class is a static container, or namespace, for a collection of related Entity Component Systems (ECS) that govern Non-Player Character (NPC) interactions. It does not hold state or contain logic itself. Instead, it groups systems that work in concert to initialize and manage the interaction capabilities of NPCs.

This class embodies the ECS principle of separating data (Components) from logic (Systems). The nested classes within are discrete, single-responsibility systems that are registered with the server's main simulation loop.

---
description: Architectural reference for NPCInteractionSystems.AddSimulationManagerSystem
---

# NPCInteractionSystems.AddSimulationManagerSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public static class AddSimulationManagerSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
This system acts as an **initialization and bootstrapping agent** within the server's ECS framework. Its sole responsibility is to ensure that any entity classified as an NPC is properly equipped with an InteractionManager component. It operates reactively, listening for the creation of new entities that fit a specific profile.

The system's logic is driven by its Query, which targets entities that possess an NPCEntity component but *do not yet* have an InteractionManager. This precise targeting prevents redundant operations on already-initialized entities and makes the system idempotent.

Upon detecting a qualifying entity, the system instantiates a new InteractionManager, configured with a specialized NPCInteractionSimulationHandler, and attaches it to the entity. This effectively "activates" the NPC's ability to be processed by other interaction-related systems, such as the TickHeldInteractionsSystem.

## Lifecycle & Ownership
- **Creation:** Instantiated once by a System Registry or World Initializer during server bootstrap. It is not created on a per-entity basis.
- **Scope:** Singleton-per-world. It persists for the entire lifetime of the world simulation it is registered with.
- **Destruction:** Decommissioned and garbage collected only when the parent world or server instance is shut down.

## Internal State & Concurrency
- **State:** This system is **stateless**. It holds immutable references to ComponentType definitions, which are configured at construction and never change. It does not cache entity data or maintain any state between invocations.
- **Thread Safety:** The system is inherently thread-safe due to its stateless nature. The ECS framework guarantees that the onEntityAdd method is invoked in a controlled manner, typically from a single main thread or within a well-defined stage of the game loop, preventing race conditions on component addition.

## API Surface
The public contract is with the ECS framework, not with application-level code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Framework callback. Attaches an InteractionManager to a newly added entity that matches the query. |
| getQuery() | Query | O(1) | Framework callback. Returns the query that defines which entities this system operates on. |

## Integration Patterns

### Standard Usage
This system is not invoked directly. It is registered with the world's system manager, which then automatically invokes its callbacks as entities are created.

```java
// Example of how the engine would register this system
World world = ...;
ComponentType<EntityStore, NPCEntity> npcType = ...;

// The system is created once and registered
AddSimulationManagerSystem system = new AddSimulationManagerSystem(npcType);
world.getSystemRegistry().register(system);
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Never call onEntityAdd directly. Doing so bypasses the ECS framework's state management and will lead to unpredictable behavior.
- **Redundant Registration:** Registering multiple instances of this system for the same world will cause unnecessary overhead, though its idempotent design may prevent catastrophic failure.

## Data Pipeline
This system acts as a reactive data transformer at the point of entity creation.

> Flow:
> Entity Creation with NPCEntity component -> ECS Framework dispatches event -> **AddSimulationManagerSystem.onEntityAdd** -> holder.addComponent() -> Entity is mutated to include InteractionManager component

---
description: Architectural reference for NPCInteractionSystems.TickHeldInteractionsSystem
---

# NPCInteractionSystems.TickHeldInteractionsSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public static class TickHeldInteractionsSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
This system is a core component of the **continuous interaction simulation** for NPCs. As an EntityTickingSystem, it is invoked by the main game loop for every simulation tick, operating on a batch of relevant entities.

Its primary function is to trigger and process interactions related to items that an NPC is holding or wearing. This includes items in the main hand, off-hand, and every armor slot. The system queries for entities that have *both* an NPCEntity component and an InteractionManager, ensuring it only operates on fully initialized NPCs. This creates an implicit dependency on the AddSimulationManagerSystem having run first.

For each entity, it delegates the complex logic of checking and executing interactions to the entity's InteractionManager component by calling tryRunHeldInteraction. This is a critical design pattern: the system is the *driver* that initiates the process, while the component holds the *state and logic* specific to that entity. The use of a CommandBuffer for state changes ensures that all modifications to the world state are deferred and executed at a safe synchronization point at the end of the tick, preventing concurrency issues.

## Lifecycle & Ownership
- **Creation:** Instantiated once by a System Registry or World Initializer during server bootstrap.
- **Scope:** Singleton-per-world. It persists for the entire lifetime of the world simulation.
- **Destruction:** Decommissioned and garbage collected when the parent world or server instance is shut down.

## Internal State & Concurrency
- **State:** This system is **stateless**. It does not maintain any data between ticks. All state is read directly from entity components during the tick method.
- **Thread Safety:** The ECS scheduler is responsible for ensuring thread safety. The tick method may be executed concurrently for different chunks of entities on different worker threads. The system's design supports this by being stateless and deferring all world-mutating operations to the provided CommandBuffer. This is a fundamental pattern for achieving scalable, parallelized game logic.

## API Surface
The public contract is with the ECS scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, buffer) | void | O(N) | Framework callback. Invoked each tick to process held-item interactions for a chunk of N entities. |
| getQuery() | Query | O(1) | Framework callback. Returns the query that defines which entities this system operates on. |

## Integration Patterns

### Standard Usage
Like its sibling system, TickHeldInteractionsSystem is registered with the world's system manager and is never called directly.

```java
// Example of how the engine would register this system
World world = ...;
ComponentType<EntityStore, NPCEntity> npcType = ...;

// The system is created once and registered to run every tick
TickHeldInteractionsSystem system = new TickHeldInteractionsSystem(npcType);
world.getSystemRegistry().register(system);
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Calling the tick method directly is a severe violation of the ECS architecture. It circumvents the scheduler, the CommandBuffer synchronization, and will corrupt world state.
- **Incorrect Dependencies:** Attempting to run this system on an entity that does not have an InteractionManager will result in assertion failures and crashes. The query correctly prevents this, but manual manipulation of components could bypass this safeguard.

## Data Pipeline
This system reads entity state and produces commands to change world state.

> Flow:
> Game Loop Tick -> ECS Scheduler -> **TickHeldInteractionsSystem.tick** -> Reads NPCEntity and InteractionManager components -> Calls InteractionManager.tryRunHeldInteraction -> Writes potential state changes to CommandBuffer -> End of Tick Synchronization Point

