---
description: Architectural reference for ObjectivePlayerSetupSystem
---

# ObjectivePlayerSetupSystem

**Package:** com.hypixel.hytale.builtin.adventure.objectives.systems
**Type:** System Component

## Definition
```java
// Signature
public class ObjectivePlayerSetupSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
The ObjectivePlayerSetupSystem is a reactive, event-driven component within Hytale's Entity Component System (ECS). Its sole responsibility is to configure the objective state for a player entity at the precise moment it is introduced into a world. This system acts as the critical bridge between the generic player lifecycle (spawning, joining) and the specialized Adventure Mode objective module.

It operates based on a query that targets entities possessing both a **Player** component and a **UUIDComponent**. When the ECS engine detects a new entity matching this signature, it automatically invokes this system's logic for that entity.

The system's core functions are twofold:
1.  **State Restoration:** For returning players, it reads their persistent configuration data and re-enrolls them in any objectives that were active when they last left the world. This ensures continuity of quests across sessions.
2.  **Initial State Provisioning:** For players spawning in a world for the very first time, it consults the world's gameplay configuration to trigger a designated "starter" objective line. This serves as the primary mechanism for onboarding players into a world's narrative or progression.

Crucially, this system also guarantees the presence of an **ObjectiveHistoryComponent** on every player entity, which is a foundational component required by other objective-related systems.

## Lifecycle & Ownership
- **Creation:** A single instance of ObjectivePlayerSetupSystem is instantiated by the ECS framework during server or world initialization. It is registered as part of the objective module's services and is not intended for manual creation.
- **Scope:** The system is a long-lived, stateless service. Its lifecycle is tied to the world it operates on; it persists as long as the world is loaded and running.
- **Destruction:** The instance is destroyed and garbage collected when the world is unloaded or the server shuts down. It performs no explicit cleanup on its own removal.

## Internal State & Concurrency
- **State:** This system is entirely **stateless**. Its fields are immutable references to component definitions, configured at construction via dependency injection. It holds no per-player data or mutable state between invocations. All necessary data is read directly from entity components or global configuration during execution.
- **Thread Safety:** This system is **not thread-safe** and is designed to be executed exclusively by the main server game loop thread. Its operations, particularly the use of a CommandBuffer, rely on the single-threaded, transactional nature of the ECS tick cycle. Invoking its methods from an external thread will lead to state corruption, race conditions, and unpredictable behavior.

## API Surface
The public methods of this class are framework callbacks and are not intended for direct developer invocation. The primary interaction with this system is implicit, by creating an entity that matches its query.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the entity query that this system subscribes to. Invoked by the ECS engine. |
| onEntityAdded(ref, reason, store, commandBuffer) | void | O(N) | **Framework Hook.** The core logic handler, triggered when a new player entity enters the world. Complexity is O(N) where N is the number of active objectives for the player. |
| onEntityRemove(ref, reason, store, commandBuffer) | void | O(1) | **Framework Hook.** Triggered when a player entity is removed. This implementation is empty, as cleanup is handled by other systems. |

## Integration Patterns

### Standard Usage
A developer does not call methods on this system directly. Interaction is implicit: the system automatically processes any entity that is created with the required components. The engine handles the invocation.

The following code demonstrates how to create a player entity that would be processed by this system.

```java
// This code would be executed within another system or entity creation logic.
// The ObjectivePlayerSetupSystem will AUTOMATICALLY be triggered by the engine
// after this entity is committed to the world.

// Assume 'commandBuffer' is a valid CommandBuffer<EntityStore>
// Assume 'playerUUID' is the UUID of the new player

Ref<EntityStore> newPlayerEntity = commandBuffer.createEntity();
commandBuffer.addComponent(newPlayerEntity, new Player(...));
commandBuffer.addComponent(newPlayerEntity, new UUIDComponent(playerUUID));

// No further action is needed. The system will now handle objective setup.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ObjectivePlayerSetupSystem()`. The ECS engine is responsible for its lifecycle and for injecting its dependencies. Manual creation will result in a non-functional object.
- **Manual Invocation:** Do not call `onEntityAdded` directly. This bypasses the transactional CommandBuffer and the engine's update cycle, which will break state consistency and cause severe bugs. The method must only be called by the ECS engine.
- **Partial Entity Creation:** Creating an entity with a Player component but without a UUIDComponent (or vice-versa) will cause the entity to be ignored by this system, leaving the player in an uninitialized objective state.

## Data Pipeline
The system is triggered by an entity creation event and orchestrates a flow of data from persistent storage to the live entity state.

> Flow:
> Player Joins Server -> ECS Engine creates Player Entity -> **ObjectivePlayerSetupSystem** is triggered by its Query -> System reads `PlayerConfigData` (persistent state) -> System writes `ObjectiveHistoryComponent` to the entity via CommandBuffer -> System calls `ObjectivePlugin` to start or resume objectives -> `ObjectivePlugin` adds new objective tracking components to the Player Entity.

