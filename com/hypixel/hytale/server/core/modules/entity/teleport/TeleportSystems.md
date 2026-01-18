---
description: Architectural reference for TeleportSystems
---

# TeleportSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.teleport
**Type:** Utility

## Definition
```java
// Signature
public class TeleportSystems {
    public static class MoveSystem extends RefChangeSystem<EntityStore, Teleport> { ... }
    public static class PlayerMoveCompleteSystem extends RefChangeSystem<EntityStore, PendingTeleport> { ... }
    public static class PlayerMoveSystem extends RefChangeSystem<EntityStore, Teleport> { ... }
}
```

## Architecture & Concepts

The TeleportSystems class is a container for a group of related systems that implement entity teleportation logic within the server's Entity Component System (ECS) framework. It is not a concrete object but a namespace for three distinct, highly-specialized systems that react to component changes on entities.

The core architectural pattern employed here is **Command as a Component**. The `Teleport` component is not a persistent state but a transient command. When another part of the game logic wishes to teleport an entity, it adds a `Teleport` component to it. The systems within TeleportSystems are designed to detect, process, and then immediately remove this command component, executing the teleport as a side effect.

This design cleanly separates the *intent* to teleport from the *implementation*, which differs significantly between players and non-player entities.

*   **MoveSystem:** Handles the simple, immediate teleportation of non-player entities (e.g., NPCs, creatures).
*   **PlayerMoveSystem & PlayerMoveCompleteSystem:** Work in tandem to manage the more complex, two-phase teleportation process for players, which requires synchronization with the game client. This creates a state machine managed by components.

## Lifecycle & Ownership

-   **Creation:** The nested system classes (MoveSystem, PlayerMoveSystem, etc.) are instantiated by the server's core ECS engine during the server bootstrap or world initialization phase. They are registered with the ECS runner which invokes them during the game loop.
-   **Scope:** These systems persist for the entire lifetime of the world or server they are registered with. They are stateless services that operate on component data.
-   **Destruction:** The systems are destroyed when the server or world shuts down and the ECS engine is dismantled.

The components these systems operate on, `Teleport` and `PendingTeleport`, are extremely short-lived. A `Teleport` component typically exists for a single tick before being processed and removed. A `PendingTeleport` component exists for the duration of the client-server network round-trip.

## Internal State & Concurrency

-   **State:** The TeleportSystems classes are **stateless**. All state they operate on is external, residing within the components attached to entities (e.g., `Teleport`, `TransformComponent`). They do not cache data between invocations.
-   **Thread Safety:** These systems are **not thread-safe** and are designed to be executed within a single-threaded, per-world game loop. The ECS framework guarantees that they are called in a deterministic order. All structural modifications to the entity store (adding/removing components or entities) are deferred via the `CommandBuffer` parameter. This is a critical mechanism to prevent race conditions and maintain world state integrity during a tick.

**WARNING:** Never invoke system methods like onComponentAdded directly from multiple threads. All interactions must be mediated by adding components to the world's entity store.

## API Surface

The public contract of these systems is not a set of callable methods, but rather their reaction to component changes. Direct invocation is an anti-pattern.

| System Class | Trigger Component | Scope | Description |
| :--- | :--- | :--- | :--- |
| **MoveSystem** | `Teleport` | Non-Player Entities | Reacts to the addition of a `Teleport` component. Instantly updates the entity's `TransformComponent`, handles cross-world entity migration, and removes the `Teleport` component. |
| **PlayerMoveSystem** | `Teleport` | Player Entities | Initiates the two-phase player teleport. Sends a `ClientTeleport` packet, updates server-side transforms, and replaces the `Teleport` component with a `PendingTeleport` component to await client confirmation. |
| **PlayerMoveCompleteSystem** | `PendingTeleport` | Player Entities | Reacts to the *removal* of a `PendingTeleport` component (which signifies client acknowledgement). Finalizes the teleport by updating physics-related components like `CollisionResultComponent`. |

## Integration Patterns

### Standard Usage

To teleport any entity, add a `Teleport` component to it via a `CommandBuffer`. The appropriate system will automatically be invoked by the ECS engine on the next tick.

```java
// How a developer should teleport an entity
// Assume 'entityRef' and 'commandBuffer' are in scope
Vector3d newPosition = new Vector3d(100, 64, 100);
Vector3f newRotation = new Vector3f(0, 90, 0);

// Create the command component
Teleport teleportCommand = new Teleport(newPosition, newRotation);

// Add the command to the entity. The system will handle the rest.
commandBuffer.addComponent(entityRef, teleportCommand);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new PlayerMoveSystem()`. The systems are managed entirely by the ECS framework.
-   **Manual Invocation:** Never call `onComponentAdded` or other system methods directly. This bypasses the `CommandBuffer` and the entire ECS lifecycle, which will corrupt world state and cause severe concurrency issues.
-   **State Manipulation:** Do not manually add a `PendingTeleport` component to a player. This component is an internal implementation detail of the two-phase teleport state machine and should only be managed by `PlayerMoveSystem`.

## Data Pipeline

The data flow for teleportation is fundamentally different for non-players versus players.

**Non-Player Entity Teleport Flow:**

> Flow:
> Game Logic -> `CommandBuffer.addComponent(Teleport)` -> **MoveSystem** -> Updates `TransformComponent` -> `CommandBuffer.removeComponent(Teleport)`

**Player Entity Teleport Flow (Two-Phase):**

> **Phase 1: Initiation**
> Flow:
> Game Logic -> `CommandBuffer.addComponent(Teleport)` -> **PlayerMoveSystem** -> Sends `ClientTeleport` Packet -> `CommandBuffer.addComponent(PendingTeleport)` -> `CommandBuffer.removeComponent(Teleport)`

> **Phase 2: Completion**
> Flow:
> Client Acknowledges Teleport -> Network Handler -> `CommandBuffer.removeComponent(PendingTeleport)` -> **PlayerMoveCompleteSystem** -> Finalizes `CollisionResultComponent` State

