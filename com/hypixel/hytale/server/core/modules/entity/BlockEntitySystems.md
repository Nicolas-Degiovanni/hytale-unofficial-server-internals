---
description: Architectural reference for BlockEntitySystems.BlockEntitySetupSystem
---

# BlockEntitySystems.BlockEntitySetupSystem

**Package:** com.hypixel.hytale.server.core.modules.entity
**Type:** System Component

## Definition
```java
// Signature
public static class BlockEntitySetupSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
BlockEntitySetupSystem is a foundational system within the Entity Component System (ECS) framework, responsible for the one-time initialization of entities that possess a BlockEntity component. It operates as a reactive "setup" system, triggered by the ECS framework immediately upon the addition of a matching entity to the world.

Its primary architectural role is to enforce component contracts and dependencies. When a new BlockEntity is created, this system guarantees that it is correctly integrated into other core engine systems, such as networking and physics. It achieves this by adding and configuring essential components like NetworkId and BoundingBox, effectively bootstrapping the entity into a valid, processable state.

This system decouples the initial state of a BlockEntity prefab or archetype from its final, runtime state, allowing for dynamic and context-aware initialization.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's core module loader during the bootstrap sequence of an EntityStore (e.g., world creation). It is then registered with the ECS Store's system manager.
- **Scope:** Session-scoped. The system instance persists for the entire lifetime of the EntityStore it is registered with.
- **Destruction:** The system is discarded and eligible for garbage collection only when its parent EntityStore is destroyed, for example, during a world unload or server shutdown.

## Internal State & Concurrency
- **State:** This system is stateless. It holds an immutable reference to the ComponentType for BlockEntity but does not maintain any per-entity state between invocations. All operations are performed on the components of the entity being processed.
- **Thread Safety:** The ECS framework guarantees that onEntityAdd is invoked in a thread-safe context, typically during a specific, non-parallel phase of the world update cycle where entity structural changes are processed. Direct concurrent invocation from application code is not supported and will lead to undefined behavior.

## API Surface
The public contract is implicitly defined by its superclass, HolderSystem. The engine invokes its lifecycle methods; they are not intended for direct user calls.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | **Engine Callback.** Triggered when an entity matching the query is added. Initializes NetworkId and BoundingBox components. |
| getQuery() | Query | O(1) | Defines the component signature that triggers this system, which is any entity possessing a BlockEntity component. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. Its execution is a side effect of creating an entity with a BlockEntity component. The framework handles the invocation automatically.

```java
// A developer adds a BlockEntity component to an entity.
// The ECS framework detects this and automatically invokes BlockEntitySetupSystem.
Entity newEntity = store.createEntity();
commandBuffer.addComponent(newEntity.getRef(), BlockEntity.getComponentType(), new MyBlockEntity());
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Never call onEntityAdd directly. This bypasses the ECS framework's state management and will lead to component desynchronization and world corruption.
- **Component Dependency Violation:** Creating a BlockEntity and then immediately attempting to access its BoundingBox within the same command buffer sequence is a race condition. The BoundingBox is added by this system in a subsequent processing phase.

## Data Pipeline
The system acts as an initializer in the entity creation pipeline.

> Flow:
> CommandBuffer.addComponent(BlockEntity) -> ECS Framework processes commands -> **BlockEntitySetupSystem.onEntityAdd** -> Holder.addComponent(NetworkId) & Holder.putComponent(BoundingBox) -> Entity is now fully initialized

---
description: Architectural reference for BlockEntitySystems.BlockEntityTrackerSystem
---

# BlockEntitySystems.BlockEntityTrackerSystem

**Package:** com.hypixel.hytale.server.core.modules.entity
**Type:** System Component

## Definition
```java
// Signature
public static class BlockEntityTrackerSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
BlockEntityTrackerSystem is a critical component of the server's state replication and networking layer. As an EntityTickingSystem, it runs every game tick, but its function is not to simulate game logic. Instead, its sole responsibility is to synchronize the state of BlockEntity instances with connected clients.

The system queries for all entities that are both a BlockEntity and currently visible to at least one player (as indicated by the EntityTrackerSystems.Visible component). It performs change detection by checking network-dirty flags on components like BlockEntity and EntityScaleComponent.

If a change is detected, or if an entity becomes newly visible to a player, the system serializes the relevant state into a ComponentUpdate packet and queues it for delivery to the appropriate clients via the EntityViewer object. This architecture ensures that network bandwidth is used efficiently, as updates are only sent when state changes or visibility changes occur.

### Lifecycle & Ownership
- **Creation:** Instantiated once during the server's primary ECS initialization for a given EntityStore.
- **Scope:** Persists for the entire lifetime of the EntityStore.
- **Destruction:** Destroyed when the parent EntityStore is shut down.

## Internal State & Concurrency
- **State:** This system is stateless. It does not store data between ticks. All decisions are based on the current state of the components it queries.
- **Thread Safety:** This system is designed for parallel execution. The isParallel method indicates that the ECS scheduler may distribute the workload across multiple threads. The framework guarantees safety by partitioning the set of entities (via ArchetypeChunks) so that each thread works on a disjoint set. The final call to viewer.queueUpdate must be a thread-safe operation, as multiple worker threads may call it concurrently for different entities visible to the same player.

## API Surface
The public contract is implicitly defined by its superclass, EntityTickingSystem.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, commands) | void | O(N) | **Engine Callback.** Executed each tick for every matching entity. Checks for state changes and queues network updates. N is the number of viewers. |
| getQuery() | Query | O(1) | Defines the query for entities that are visible and are a BlockEntity. |
| getGroup() | SystemGroup | O(1) | Assigns this system to the QUEUE_UPDATE_GROUP, ensuring it runs after visibility calculations but before network packets are flushed. |

## Integration Patterns

### Standard Usage
To trigger a network update for a BlockEntity, another system must modify its state and explicitly mark it as dirty for the network.

```java
// In another system...
BlockEntity blockEntity = holder.getComponent(BlockEntity.getComponentType());

// Modify the state
blockEntity.setBlockTypeKey("hytale:stone_pillar");

// The setBlockTypeKey method internally sets a network-dirty flag.
// BlockEntityTrackerSystem will automatically detect this on the next tick and send the update.
```

### Anti-Patterns (Do NOT do this)
- **Forgetting to Dirty:** Modifying the internal state of a BlockEntity or EntityScaleComponent without calling the proper setter method that flags it for network updates. This will result in a "ghost" change, where the server state is updated but clients are never informed.
- **Direct Packet Creation:** Bypassing this system to manually create and send ComponentUpdate packets. This breaks the engine's change detection and visibility tracking, leading to inconsistent state and wasted bandwidth.

## Data Pipeline
This system bridges the gap between game state and the network protocol.

> Flow:
> Game Logic dirties a BlockEntity component -> **BlockEntityTrackerSystem.tick** detects dirty flag -> A ComponentUpdate packet is created -> packet is queued on the viewer's update list -> Network Flush System -> TCP/UDP Packet -> Client

---
description: Architectural reference for BlockEntitySystems.Ticking
---

# BlockEntitySystems.Ticking

**Package:** com.hypixel.hytale.server.core.modules.entity
**Type:** System Component

## Definition
```java
// Signature
public static class Ticking extends EntityTickingSystem<EntityStore> implements DisableProcessingAssert {
```

## Architecture & Concepts
This system is the primary driver for the physics and continuous logic updates of BlockEntity instances. As an EntityTickingSystem, it is invoked by the ECS scheduler every game tick for all entities that match its strict archetype: they must possess a TransformComponent, a BlockEntity, and a Velocity component.

Its core architectural function is to delegate. It does not contain the physics simulation logic itself. Instead, it retrieves the SimplePhysicsProvider associated with each BlockEntity and invokes its tick method. This pattern allows different types of BlockEntity to implement unique physics behaviors while centralizing the update loop and error handling.

A key feature is its robust exception handling. If a physics provider throws an unhandled exception during its tick, this system catches it, logs a detailed error, and issues a command to remove the offending entity. This prevents a single faulty entity from crashing the entire server, ensuring high availability. The implementation of the DisableProcessingAssert marker interface suggests it opts out of certain engine-level safety checks, taking full responsibility for its entities' lifecycle.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's core module loader when an EntityStore is initialized.
- **Scope:** Lives for the duration of the EntityStore.
- **Destruction:** Discarded when the parent EntityStore is destroyed.

## Internal State & Concurrency
- **State:** This system is entirely stateless. It processes each entity independently based on the entity's own component data.
- **Thread Safety:** The system's design is compatible with parallel execution, though it does not explicitly request it. The ECS scheduler is responsible for invoking the tick method in a thread-safe manner, typically by processing disjoint sets of entities (ArchetypeChunks) on different threads. All state modifications are performed through the CommandBuffer or by directly mutating components on the chunk, which is safe within this execution model.

## API Surface
The public contract is implicitly defined by its superclass, EntityTickingSystem.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, commands) | void | O(C) | **Engine Callback.** Executed each tick for every matching entity. Delegates to the entity's physics provider. C is the complexity of the physics tick. |
| getQuery() | Query | O(1) | Defines the query for entities that have Transform, BlockEntity, and Velocity components. |

## Integration Patterns

### Standard Usage
A developer enables physics on an entity by ensuring it has the three required components. The system is entirely automatic after that.

```java
// Create an entity with all necessary components for physics processing.
Entity newEntity = store.createEntity();
commandBuffer.addComponent(newEntity.getRef(), TransformComponent.getComponentType(), new TransformComponent());
commandBuffer.addComponent(newEntity.getRef(), Velocity.getComponentType(), new Velocity());
commandBuffer.addComponent(newEntity.getRef(), BlockEntity.getComponentType(), new MyPhysicsEnabledBlockEntity());

// The Ticking system will now automatically process this entity every tick.
```

### Anti-Patterns (Do NOT do this)
- **Incomplete Archetype:** Creating a BlockEntity with a TransformComponent but no Velocity component (or vice-versa). The entity will not be processed by this system, and its physics will not update.
- **Unstable Physics Provider:** Implementing a SimplePhysicsProvider with logic that can enter an infinite loop or throw frequent exceptions. While the system's error handling will prevent a server crash, it will cause entities to unexpectedly disappear from the world.

## Data Pipeline
This system is a core consumer of time (delta time) and a mutator of positional and velocity state.

> Flow:
> Main Game Loop -> ECS Scheduler invokes systems -> **Ticking.tick** -> SimplePhysicsProvider.tick -> Physics simulation updates TransformComponent and Velocity -> State is ready for the next tick's processing and network replication.

