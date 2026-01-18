---
description: Architectural reference for InteractionSystems.PlayerAddManagerSystem
---

# InteractionSystems.PlayerAddManagerSystem

**Package:** com.hypixel.hytale.server.core.modules.interaction.system
**Type:** System Component

## Definition
```java
// Signature
public static class PlayerAddManagerSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The PlayerAddManagerSystem is a foundational system within the server-side Entity Component System (ECS) framework. Its sole responsibility is to ensure that every entity with a Player component is also equipped with an InteractionManager component.

This system operates as a reactive bootstrap mechanism. It queries the ECS world for entities that are players but lack an InteractionManager. Upon detection, it immediately attaches a new InteractionManager instance to the entity, making it capable of processing and simulating interactions. This pattern decouples the creation of a player entity from the initialization of its interaction capabilities, promoting modularity.

## Lifecycle & Ownership
- **Creation:** Instantiated once by the server's primary ECS world builder, typically during the setup phase of the InteractionModule. It is registered within the world's system graph to listen for entity additions.
- **Scope:** Session-scoped. It persists for the entire lifetime of the server's ECS world.
- **Destruction:** The system is destroyed and garbage collected when the server world is unloaded or during a full server shutdown.

## Internal State & Concurrency
- **State:** This system is stateless. Its behavior is driven entirely by the state of the entities it processes. The internal query object is immutable after construction.
- **Thread Safety:** The onEntityAdd method is invoked by the ECS framework's main processing loop. It is not designed to be called directly from arbitrary threads. All component modifications are made via the Holder proxy, which ensures that changes are safely queued within the current ECS transaction.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Framework-invoked callback. Attaches a new InteractionManager to the entity represented by the holder. |
| getQuery() | Query | O(1) | Returns the ECS query that identifies player entities lacking an InteractionManager. Used by the framework. |

## Integration Patterns

### Standard Usage
This system is not used directly by developers. It is automatically registered with the ECS world by the InteractionModule. Its operation is transparent.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this system using `new`. The ECS framework manages its lifecycle.
- **Manual Invocation:** Do not call onEntityAdd directly. Doing so would bypass the ECS world's state management and transactional integrity, leading to unpredictable behavior and state corruption.

## Data Pipeline
> Flow:
> New Entity Created -> Player Component Added -> ECS Query Match -> **PlayerAddManagerSystem.onEntityAdd** -> InteractionManager Component Added to Entity

---
description: Architectural reference for InteractionSystems.TickInteractionManagerSystem
---

# InteractionSystems.TickInteractionManagerSystem

**Package:** com.hypixel.hytale.server.core.modules.interaction.system
**Type:** System Component

## Definition
```java
// Signature
public static class TickInteractionManagerSystem extends EntityTickingSystem<EntityStore> implements EntityStatsSystems.StatModifyingSystem {
```

## Architecture & Concepts
This system is the core engine for processing active interactions on the server. As an EntityTickingSystem, it is invoked by the main game loop for every entity that possesses an InteractionManager component.

Its primary responsibilities are:
1.  **Simulation:** It delegates the per-tick update logic to the InteractionManager component itself by calling its tick method. This advances the state of any ongoing interactions.
2.  **Network Synchronization:** After ticking, it checks the InteractionManager for any new network packets (SyncInteractionChain) that need to be sent to the client. It serializes and dispatches these packets to ensure the client's view of the interaction state is consistent with the server.
3.  **Error Handling:** The entire tick logic for an entity is wrapped in a robust try-catch block. If any exception occurs during interaction processing, it is logged, and the offending entity is removed from the world to prevent a single faulty entity from crashing the entire server.
4.  **Execution Ordering:** By belonging to the DamageModule's `gatherDamageGroup`, its execution is explicitly ordered relative to other critical game systems, such as damage calculation.

## Lifecycle & Ownership
- **Creation:** Instantiated by the ECS world builder when the InteractionModule is initialized.
- **Scope:** Persists for the lifetime of the server world.
- **Destruction:** Decommissioned when the server world is shut down.

## Internal State & Concurrency
- **State:** This system is stateless. All state is contained within the components it operates on (e.g., InteractionManager, PlayerRef).
- **Thread Safety:** The ECS scheduler may execute this system's tick method in parallel across different chunks of entities. Operations on the CommandBuffer are inherently safe as they queue commands for later execution. Network writes via the PlayerRef must be thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(N) | The main update logic, called once per entity per game tick. N is the complexity of the interaction simulation. |
| getGroup() | SystemGroup | O(1) | Declares its membership in a specific system group to enforce execution order. |

## Integration Patterns

### Standard Usage
This system is automatically registered and managed by the ECS framework. Its functionality is invoked as part of the server's main game loop.

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Calling the tick method manually will break the server's update cycle, bypass concurrency controls, and corrupt game state.
- **State Storage:** Do not add mutable fields to this system. It must remain stateless to support parallel execution.

## Data Pipeline
> Flow:
> Game Tick -> ECS Scheduler -> **TickInteractionManagerSystem.tick** -> InteractionManager.tick() -> State Change -> SyncInteractionChain Packet Created -> PlayerRef Packet Handler -> Network Layer

---
description: Architectural reference for InteractionSystems.TrackerTickSystem
---

# InteractionSystems.TrackerTickSystem

**Package:** com.hypixel.hytale.server.core.modules.interaction.system
**Type:** System Component

## Definition
```java
// Signature
public static class TrackerTickSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The TrackerTickSystem is a network synchronization system that bridges the Interaction system with the EntityTracker system. Its purpose is to efficiently notify clients when the set of available interactions for an entity has changed.

It operates on entities that have both an Interactions component and are visible to at least one player (tracked by the EntityTrackerSystems.Visible component). Each tick, it checks a dirty flag on the Interactions component. If the flag is set, or if new players have just gained visibility of the entity, the system serializes the entity's current interaction data into a ComponentUpdate packet. This packet is then queued on the EntityViewer for each relevant player, to be sent in the next network flush.

This system is critical for UI elements on the client, ensuring that interaction prompts (e.g., "Press E to Open") are always accurate.

## Lifecycle & Ownership
- **Creation:** Instantiated and registered by the ECS world builder.
- **Scope:** Lives for the duration of the server world.
- **Destruction:** Destroyed when the server world unloads.

## Internal State & Concurrency
- **State:** This system is stateless.
- **Thread Safety:** Designed for parallel execution, as indicated by the `isParallel` method. It reads from components and queues data onto the EntityViewer, which are thread-safe operations within the engine's architecture.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, cmd) | void | O(V) | Called per-entity per-tick. If interactions are outdated, it queues updates for all V viewers. |
| getGroup() | SystemGroup | O(1) | Belongs to the QUEUE_UPDATE_GROUP, ensuring it runs before the main network packet flush. |

## Integration Patterns

### Standard Usage
This system is an internal part of the interaction and entity tracking modules. It is registered automatically and requires no direct developer interaction.

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Calling tick directly will desynchronize game state and bypass the entity tracking framework.

## Data Pipeline
> Flow:
> Interactions Component Changed -> `consumeNetworkOutdated()` flag set -> **TrackerTickSystem.tick** -> ComponentUpdate Packet Created -> EntityViewer.queueUpdate() -> Network Flush

---
description: Architectural reference for InteractionSystems.EntityTrackerRemove
---

# InteractionSystems.EntityTrackerRemove

**Package:** com.hypixel.hytale.server.core.modules.interaction.system
**Type:** System Component

## Definition
```java
// Signature
public static class EntityTrackerRemove extends RefChangeSystem<EntityStore, Interactions> {
```

## Architecture & Concepts
This system is a reactive cleanup handler responsible for maintaining client-server state consistency. It is a RefChangeSystem that specifically listens for the *removal* of the Interactions component from any entity.

When an entity loses its Interactions component, any client that can see that entity must be notified so it can remove the corresponding UI and interaction handlers. This system accomplishes this by iterating through all players tracking the entity (its "viewers") and queuing a special "remove" command for the Interactions component type. This prevents clients from having "ghost" interactions for entities that are no longer interactive.

## Lifecycle & Ownership
- **Creation:** Instantiated and registered by the ECS world builder.
- **Scope:** Lives for the duration of the server world.
-- **Destruction:** Destroyed when the server world unloads.

## Internal State & Concurrency
- **State:** This system is stateless.
- **Thread Safety:** The onComponentRemoved method is invoked by the ECS framework. It is guaranteed to run in a context where it is safe to access the entity's components and queue network updates.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentRemoved(ref, comp, store, cmd) | void | O(V) | Framework callback. Queues a component removal update for all V viewers of the entity. |

## Integration Patterns

### Standard Usage
This is an internal system managed by the ECS framework. Its operation is transparent.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create instances of this system manually.
- **Manual Invocation:** Calling onComponentRemoved directly will lead to inconsistent state and is not supported.

## Data Pipeline
> Flow:
> Interactions Component Removed from Entity -> ECS Framework -> **EntityTrackerRemove.onComponentRemoved** -> EntityViewer.queueRemove() -> Network Flush -> Client removes interactions

---
description: Architectural reference for InteractionSystems.CleanUpSystem
---

# InteractionSystems.CleanUpSystem

**Package:** com.hypixel.hytale.server.core.modules.interaction.system
**Type:** System Component

## Definition
```java
// Signature
public static class CleanUpSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The CleanUpSystem is a critical resource management system that prevents memory and state leaks related to interactions. It hooks into the entity destruction lifecycle.

Its sole function is to listen for the removal of any entity that has an InteractionManager component. When such an entity is being destroyed, this system's onEntityRemove method is invoked. It retrieves the InteractionManager and calls its `clear` method. This action is responsible for terminating any active interaction simulations, releasing references, and ensuring the component is in a clean state before being garbage collected. This prevents dangling state or resource handles associated with an entity that no longer exists.

## Lifecycle & Ownership
- **Creation:** Instantiated and registered by the ECS world builder.
- **Scope:** Lives for the duration of the server world.
- **Destruction:** Destroyed when the server world unloads.

## Internal State & Concurrency
- **State:** This system is stateless.
- **Thread Safety:** The onEntityRemove method is called by the ECS framework during entity destruction. This is a synchronized part of the entity lifecycle, making the operation safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityRemove(ref, reason, store, cmd) | void | O(1) | Framework callback. Triggers the cleanup logic on the entity's InteractionManager. |

## Integration Patterns

### Standard Usage
This is an internal system managed by the ECS framework. Its operation is transparent.

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Calling onEntityRemove directly is a severe error. Entity destruction and cleanup must be managed exclusively by the ECS framework.

## Data Pipeline
> Flow:
> Entity Removed from World -> ECS Framework -> **CleanUpSystem.onEntityRemove** -> InteractionManager.clear() -> Internal State and Resources Released

