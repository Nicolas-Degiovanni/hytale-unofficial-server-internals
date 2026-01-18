---
description: Architectural reference for PlayerRefAddedSystem
---

# PlayerRefAddedSystem

**Package:** com.hypixel.hytale.server.core.universe.system
**Type:** Transient

## Definition
```java
// Signature
public class PlayerRefAddedSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The PlayerRefAddedSystem is a reactive, event-driven component within the server's Entity-Component-System (ECS) framework. Its primary architectural role is to act as a **bridge** between the low-level ECS data layer (the EntityStore) and the high-level game simulation layer (the World).

This system subscribes to state changes within the EntityStore, specifically listening for the addition or removal of entities that possess a PlayerRef component. When such an event occurs, it translates this low-level data change into a high-level, domain-specific action: registering or unregistering a player entity with the central World object.

This pattern effectively decouples the World from the underlying implementation of the EntityStore. The World does not need to poll for new players or understand the mechanics of component addition. Instead, it relies on this system to maintain the consistency of its own player tracking collections, ensuring the game state remains synchronized with the ECS state.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's central ECS SystemManager during world initialization. It is one of many systems that are registered to process entity and component changes each server tick.
- **Scope:** The lifecycle of a PlayerRefAddedSystem instance is tightly bound to the lifecycle of the World it services. It persists as long as the server world is active.
- **Destruction:** The instance is discarded and marked for garbage collection when the corresponding World is unloaded or during server shutdown. The SystemManager orchestrates this cleanup.

## Internal State & Concurrency
- **State:** This system is effectively **stateless**. Its only instance field, playerRefComponentType, is an immutable configuration dependency injected at construction. It does not cache entity data or maintain any state between invocations. All necessary information is provided via method arguments (Ref, Store) during the execution of a server tick.
- **Thread Safety:** This system is **not thread-safe** and is designed to be operated exclusively by the main server thread within the game loop. All interactions with the Store and the World object assume a single-threaded execution model, synchronized by the parent ECS framework. Unmanaged, multi-threaded access would corrupt the World's internal player lists and lead to severe state inconsistency.

## API Surface
The public methods of this class are callbacks that form a contract with the ECS framework. They are not intended for direct user invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(ref, reason, store, cmd) | void | O(log N) | Framework callback. Triggers when an entity matching the query is added. Registers the PlayerRef with the World. Complexity depends on the World's internal tracking data structure. |
| onEntityRemove(ref, reason, store, cmd) | void | O(log N) | Framework callback. Triggers when an entity matching the query is removed. Unregisters the PlayerRef from the World. |
| getQuery() | Query | O(1) | Defines the component filter (entities with PlayerRef) that activates this system. Invoked by the framework at initialization. |
| getDependencies() | Set | O(1) | Defines system execution order. RootDependency indicates it can run early in the tick without prerequisites. Invoked by the framework at initialization. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. The system is triggered implicitly by manipulating ECS components. The correct pattern is to add a PlayerRef component to an entity, which the ECS framework will detect, causing it to invoke this system automatically.

```java
// An entity representing a new player is created
Ref<EntityStore> playerEntity = world.createEntity();

// Adding the PlayerRef component is the trigger event.
// The PlayerRefAddedSystem will now be invoked by the ECS on the next tick.
commandBuffer.addComponent(playerEntity, new PlayerRef(playerData));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PlayerRefAddedSystem()`. The ECS SystemManager is solely responsible for the lifecycle of all systems. Manual creation will result in a non-functional object that is not registered with the game loop.
- **Manual Invocation:** Do not call `onEntityAdded` or `onEntityRemove` directly. This bypasses the ECS framework's transactional command buffer and state management, which will immediately corrupt the game state and cause unpredictable behavior or crashes.
- **Stateful Modification:** Do not add fields to this class to store temporary data. Systems in Hytale's ECS are designed to be stateless processors. State should be stored in components.

## Data Pipeline
This system acts as a critical link in the player-join and player-leave data flow. It ensures the entity's existence is reflected in the global world state.

**Player Join Flow:**
> External Event (Player Connects) -> Entity Creation -> CommandBuffer adds **PlayerRef** component -> ECS Tick Processor dispatches event -> **PlayerRefAddedSystem.onEntityAdded** -> World.trackPlayerRef -> Player is now globally tracked and visible to other game systems.

**Player Leave Flow:**
> External Event (Player Disconnects) -> Entity Destruction -> CommandBuffer removes **PlayerRef** component -> ECS Tick Processor dispatches event -> **PlayerRefAddedSystem.onEntityRemove** -> World.untrackPlayerRef -> Player is no longer globally tracked.

