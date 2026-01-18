---
description: Architectural reference for PlayerSystems
---

# PlayerSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** Utility / System Container

## Definition
```java
// Signature
public class PlayerSystems {
    // Contains multiple nested static System classes
}
```

## Architecture & Concepts

PlayerSystems is not a singular, instantiable class. It is a static container that groups a collection of highly specialized, related systems within the Hytale Entity Component System (ECS) framework. These nested systems collectively define the server-side behavior, lifecycle, and state management for all entities that represent a player.

Each inner class, such as PlayerAddedSystem or ProcessPlayerInput, is a distinct system that operates on entities possessing a specific combination of components, defined by its Query. The primary responsibility of this collection is to bridge the gap between low-level network packets, the abstract ECS world state, and high-level game logic for players.

These systems manage critical aspects of the player lifecycle:
*   **Initialization:** Attaching essential components and sending initial state packets when a player entity is added to the world.
*   **Runtime Logic:** Processing player input, updating positions, managing nameplates, and handling game events like death.
*   **State Synchronization:** Ensuring the player's state in the ECS (e.g., TransformComponent) is synchronized with other core server objects (e.g., PlayerRef).
*   **Cleanup:** Saving final state and broadcasting departure messages when a player entity is removed from the world.

This class acts as a central hub for player-centric logic within the server's main tick loop, ensuring that all player-related behaviors are executed in a predictable, ordered, and performant manner.

### Lifecycle & Ownership
- **Creation:** The PlayerSystems class itself is never instantiated. Its nested static system classes are instantiated by the server's ECS scheduler, typically during world initialization or module loading. They are registered with the world's System Graph to be executed each tick.
- **Scope:** Individual systems persist for the entire lifetime of the World they are registered in. They are stateless services that operate on component data.
- **Destruction:** The systems are destroyed and garbage collected when the World instance is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** The PlayerSystems container is stateless. The nested systems are designed to be stateless as well, holding no per-player data themselves. All state is stored in the Components attached to player entities (e.g., PlayerInput, TransformComponent, Player). The systems are pure processors of this component data.
- **Thread Safety:** These systems are designed to be run by a single-threaded ECS scheduler within the main server game loop. They are **not thread-safe** for direct, concurrent invocation. State mutations are managed through a CommandBuffer. Instead of modifying the world state directly, systems queue operations (like adding or setting a component) on the CommandBuffer. These commands are then applied in a synchronized batch at a later stage in the tick, preventing race conditions and ensuring deterministic state transitions.

## API Surface

The primary "API" consists of the system classes themselves, which are registered with the ECS framework. Direct interaction is rare; developers influence these systems by manipulating the components on player entities.

| System Class | Description |
| :--- | :--- |
| PlayerAddedSystem | **CRITICAL:** Handles the logic for when a player entity is fully added to the world. Initializes game mode, sends inventory and configuration packets, and triggers spawn particle effects. This is the primary "on-join" logic controller. |
| PlayerRemovedSystem | **CRITICAL:** Executes cleanup logic when a player entity is removed. Saves the player's last position, broadcasts a "player left" message, and flushes any remaining packets. |
| ProcessPlayerInput | Processes the queue of movement and action commands stored in the PlayerInput component, applying them to the entity's state. This system translates raw input into world actions. |
| UpdatePlayerRef | Synchronizes the entity's position (from TransformComponent) and head rotation back to the global PlayerRef object. This ensures non-ECS parts of the server have access to the player's current location. |
| BlockPausedMovementSystem | A specialized system that handles movement updates when the game is paused. It prevents desynchronization by issuing a Teleport component if significant movement is detected while paused. |
| NameplateRefSystem | Manages the creation and content of a player's Nameplate component based on the presence and value of their DisplayNameComponent. |
| KillFeed...EventSystem | Listens for KillFeedEvents and populates them with the correct player display name, ensuring kill messages are accurate. |

## Integration Patterns

### Standard Usage

Developers do not call these systems directly. The systems are automatically invoked by the ECS scheduler. The standard pattern is to add, remove, or modify components on a player entity to trigger the desired system logic.

```java
// Example: Spawning a player entity into the world.
// The ECS framework will automatically run PlayerAddedSystem on this entity.

// 1. Create the player entity with its core components.
Ref<EntityStore> playerEntityRef = world.createEntity();
commandBuffer.putComponent(playerEntityRef, Player.getComponentType(), new Player(...));
commandBuffer.putComponent(playerEntityRef, PlayerRef.getComponentType(), new PlayerRef(...));
commandBuffer.putComponent(playerEntityRef, TransformComponent.getComponentType(), new TransformComponent(...));

// 2. Once the command buffer is flushed, the entity is added to the store.
// 3. The ECS scheduler detects the new entity matches the PlayerAddedSystem query.
// 4. PlayerAddedSystem.onEntityAdded() is executed for the new entity.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PlayerAddedSystem()`. Systems must be managed exclusively by the ECS framework to ensure correct ordering and execution.
- **Manual Invocation:** Never call a system's `tick` or `onEntityAdded` method directly. This bypasses the CommandBuffer and dependency ordering, leading to race conditions, state corruption, and unpredictable behavior.
- **Stateful Systems:** Do not add mutable instance fields to these system classes to store per-player state. All state must reside in components to ensure correctness when running in the ECS environment.

## Data Pipeline

This section illustrates the flow of data for a common player action: moving forward.

> Flow:
> Client Input -> Network Packet (PlayerInputUpdate) -> Server PacketHandler -> **PlayerInput Component** (InputUpdate is queued) -> ECS Tick -> **ProcessPlayerInput System** (Dequeues and processes InputUpdate) -> **TransformComponent** (Position is modified) -> ECS Tick -> **UpdatePlayerRef System** (Synchronizes new position to PlayerRef) -> LegacyEntityTrackerSystems (Broadcasts new position to other clients)

