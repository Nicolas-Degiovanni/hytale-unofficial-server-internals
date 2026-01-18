---
description: Architectural reference for ObjectiveItemEntityRemovalSystem
---

# ObjectiveItemEntityRemovalSystem

**Package:** com.hypixel.hytale.builtin.adventure.objectives.systems
**Type:** System Component

## Definition
```java
// Signature
public class ObjectiveItemEntityRemovalSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The ObjectiveItemEntityRemovalSystem is a reactive, server-side system within the Entity-Component-System (ECS) framework. Its primary function is to act as a fail-safe mechanism for objectives tied to physical item entities in the game world.

This system listens for the removal of any entity that possesses an ItemComponent. When such an entity is removed, the system inspects the item's metadata. If the item is tagged as an "objective starter" or a critical quest item, the system determines the reason for its removal.

Crucially, if the item was removed for any reason *other than* being picked up by a player (e.g., despawning due to time, destruction by explosion, falling into the void), the system concludes that the associated objective is now impossible to start or complete. It then interfaces with the ObjectivePlugin to formally cancel the objective, preventing a broken or stuck state for the player. This system is a key component for ensuring the robustness of the adventure mode quest system.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's ECS runtime when a world or EntityStore is initialized. Systems like this are not created manually but are discovered and registered with a central SystemManager.
- **Scope:** The lifecycle of this system is bound to the lifecycle of the EntityStore it monitors. It persists as long as the game world is active and loaded in memory.
- **Destruction:** De-registered and garbage collected when the corresponding game world is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** This system is **stateless**. It contains no mutable instance fields and operates exclusively on the parameters passed into its methods by the ECS framework. All state is read from the provided Holder and Store arguments during method execution.
- **Thread Safety:** The system itself is inherently thread-safe due to its stateless nature. However, its methods are designed to be invoked exclusively by the main server game loop thread as part of the ECS tick.
    - **WARNING:** Manually invoking its methods from other threads will bypass engine-level locks and lead to severe concurrency issues, such as world state corruption or ConcurrentModificationExceptions.

## API Surface
The public contract is defined by its role as a HolderSystem. Direct invocation is not intended.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the query for all entities with an ItemComponent. This is how the system subscribes to relevant entity events. |
| onEntityAdd(holder, reason, store) | void | O(1) | Callback invoked when a matching entity is added. This implementation is a no-op. |
| onEntityRemoved(holder, reason, store) | void | O(1) | Core logic. Invoked when an item entity is removed. Checks item metadata and cancels the associated objective if necessary. |

## Integration Patterns

### Standard Usage
A developer does not call methods on this class directly. Instead, it is registered with the world's system manager. The engine handles the rest.

```java
// Example of how the engine would register this system
// This code is typically in a world or server initialization block.

SystemManager systemManager = world.getSystemManager();
systemManager.register(new ObjectiveItemEntityRemovalSystem());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Use:** Do not create an instance of this class to use its methods. It is meaningless without the context provided by the ECS engine's callbacks.
- **Manual Invocation:** Never call `onEntityRemoved` manually. This bypasses the engine's event queue and state management, which can desynchronize game state and lead to unpredictable behavior.

## Data Pipeline
This system acts as a terminal event handler in a data flow that begins with an in-game world event.

> Flow:
> World Event (e.g., Item Entity Despawn) -> EntityStore Update -> ECS Event Dispatcher -> **ObjectiveItemEntityRemovalSystem.onEntityRemoved** -> ObjectivePlugin.cancelObjective -> Global Objective State Change

