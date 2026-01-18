---
description: Architectural reference for PlayerCameraAddSystem
---

# PlayerCameraAddSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** System / Utility

## Definition
```java
// Signature
public class PlayerCameraAddSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The PlayerCameraAddSystem is a reactive, data-integrity system operating within the server-side Entity Component System (ECS) framework. Its sole responsibility is to enforce a critical architectural invariant: **every entity representing a player must possess a CameraManager component.**

It functions as an autonomous agent that observes the state of the entity world. By using a declarative query, it identifies entities that have a PlayerRef component but are missing the required CameraManager. This design decouples the complex process of player entity creation from the specific concerns of camera management. Systems that spawn players do not need explicit knowledge of camera mechanics; they simply add the PlayerRef component, and this system guarantees the subsequent attachment of the CameraManager.

This pattern is a form of data-driven initialization, ensuring that entities conform to a required component schema automatically and efficiently.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's central SystemRegistry or equivalent ECS bootstrap mechanism during server initialization. It is not intended for manual creation.
- **Scope:** A single instance of this system persists for the entire lifetime of the server process. It is effectively a singleton within the scope of the ECS world it services.
- **Destruction:** The instance is decommissioned and marked for garbage collection only during a full server shutdown when the parent ECS framework is dismantled.

## Internal State & Concurrency
- **State:** This system is **stateless**. It contains no instance fields and does not cache or store any data between invocations. Its behavior is exclusively determined by the static QUERY constant and the entity data passed to it by the ECS framework.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. All operations are performed on the entity Holder passed into its methods. The ECS framework is responsible for guaranteeing that calls to onEntityAdd are synchronized and that component modifications are thread-safe. Direct, concurrent invocation from application code is unsupported and would violate the execution model.

## API Surface
The public contract of this class is defined by its implementation of the HolderSystem interface, which is consumed exclusively by the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the static, predefined query used by the ECS to identify target entities. |
| onEntityAdd(holder, reason, store) | void | O(1) | Framework callback. Ensures the CameraManager component exists on the provided entity Holder. |
| onEntityRemoved(holder, reason, store) | void | O(1) | Framework callback. This is a no-op; the system takes no action on entity removal. |

## Integration Patterns

### Standard Usage
This system is not invoked directly. It is registered with the ECS framework at startup and operates automatically. A developer interacts with this system *implicitly* by creating an entity that satisfies its query.

```java
// A separate system creates a player entity.
// This action implicitly triggers PlayerCameraAddSystem.
Entity newPlayer = world.createEntity();
newPlayer.addComponent(new PlayerRef(playerUUID));

// At the end of the tick, PlayerCameraAddSystem will have automatically
// added a CameraManager component to the newPlayer entity.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PlayerCameraAddSystem()`. The ECS framework is solely responsible for its lifecycle. Doing so would create a rogue instance that is not registered to receive entity updates.
- **Manual Invocation:** Do not call `onEntityAdd` directly. This bypasses the ECS query engine and its state management, which can lead to race conditions or inconsistent entity states.

## Data Pipeline
The system is a terminal node in a data-driven workflow. It is triggered by a change in the entity component store and results in a subsequent modification to that same store.

> Flow:
> Entity Creation or Component Addition (with PlayerRef) -> ECS Framework Query Match -> **PlayerCameraAddSystem.onEntityAdd** -> EntityStore Write (adds CameraManager) -> Entity State Updated

