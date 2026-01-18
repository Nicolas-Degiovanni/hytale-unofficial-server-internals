---
description: Architectural reference for SetMemoriesCapacityInteraction
---

# SetMemoriesCapacityInteraction

**Package:** com.hypixel.hytale.builtin.adventure.memories.interactions
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class SetMemoriesCapacityInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The SetMemoriesCapacityInteraction is a server-authoritative, data-driven class that represents a discrete, instantaneous game action. It is a concrete implementation of the engine's generic Interaction System, designed specifically to modify a player's memory storage capacity.

This class is not a long-lived service or manager. Instead, it acts as a blueprint for an action that can be attached to various game objects, such as items, blocks, or trigger volumes. Its behavior is configured through a `Codec`, allowing game designers to define different capacity upgrades in data files without changing game code.

Its core responsibility is to mutate the **PlayerMemories** component on a target entity. Upon execution, it checks the entity's current capacity and, if the new capacity is greater, applies the change and triggers associated first-time-unlock events, such as sending network packets to the client and displaying notifications.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly using the `new` keyword. They are deserialized from game data files (e.g., JSON) by the server's content loading system using the provided static `CODEC`. Each unique interaction defined in the game's content will have a corresponding object instance in memory.
- **Scope:** The object instance persists for as long as the server has the corresponding game content loaded. Its lifecycle is tied to the server's content registry, not to a specific player session or world state.
- **Destruction:** The object is eligible for garbage collection when the server unloads the content pack that defines it, or during server shutdown. There are no explicit cleanup methods.

## Internal State & Concurrency
- **State:** The class holds one piece of configuration state: the integer `capacity`. This value is set once during deserialization and is treated as immutable for the lifetime of the object. The class itself does not manage or cache any dynamic game world state; all world modifications are performed through the `InteractionContext` and `CommandBuffer`.
- **Thread Safety:** This class is **not thread-safe** and must only be used by the server's main game thread. The engine guarantees that the `firstRun` method is invoked synchronously within the server's tick cycle. All entity component modifications are marshaled through a `CommandBuffer`, which queues changes to be applied deterministically, preventing race conditions and ensuring world state integrity.

## API Surface
The primary contract is with the Interaction System via method overrides, not for direct invocation by developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | **Framework-Internal.** The entry point for the interaction's logic. Modifies the target entity's PlayerMemories component via the provided CommandBuffer. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `WaitForDataFrom.Server`, signaling to the engine that this interaction is fully server-authoritative and requires no client-side data to complete. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code. Instead, it is configured within a game's content files. A game designer would define an interaction trigger on an entity and associate it with this class, specifying the desired capacity.

*Hypothetical JSON Configuration:*
```json
{
  "id": "hytale:upgrade_memories_small",
  "interaction": {
    "type": "SetMemoriesCapacityInteraction",
    "Capacity": 10
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SetMemoriesCapacityInteraction()`. The object will lack its configured `capacity` and will not be registered with the game engine. All instances must be created by the server's content loader.
- **Direct Invocation:** Do not call the `firstRun` method directly. This bypasses the Interaction System's state management, context creation, and cooldown handling, and will result in assertion errors or world state corruption.
- **Asynchronous Access:** Do not access or execute this interaction from any thread other than the main server thread.

## Data Pipeline
The flow of data and control for this interaction is managed entirely by the server's Interaction System.

> Flow:
> Player Action (e.g., right-clicks an item) -> Server receives input -> Interaction System is invoked for the target entity -> The system identifies the configured **SetMemoriesCapacityInteraction** -> `firstRun` is called with a populated `InteractionContext` -> The method accesses the `CommandBuffer` -> `PlayerMemories` component is requested and modified in the buffer -> `PlayerRef` component is used to queue network packets (`UpdateMemoriesFeatureStatus`) and chat messages -> Server tick ends -> `CommandBuffer` changes are committed to the world state -> Network packets are dispatched to the client.

