---
description: Architectural reference for ModifyInventoryInteraction
---

# ModifyInventoryInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Configuration Object

## Definition
```java
// Signature
public class ModifyInventoryInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The ModifyInventoryInteraction is a server-authoritative, data-driven class that defines a specific set of inventory manipulations. It represents a single, atomic transaction that can add, remove, or alter items and their properties within a player's inventory.

Architecturally, this class is not a service or a manager, but rather a **behavioral template**. It is designed to be defined in external configuration files (e.g., JSON assets) and deserialized at runtime by the server using its static `CODEC`. This pattern allows game designers to create complex item interactions, such as crafting recipes, tool usage, or consumable effects, without writing new Java code.

As a subclass of SimpleInstantInteraction, it executes its entire logic immediately on the server upon being triggered. The `getWaitForDataFrom` method returns `WaitForDataFrom.Server`, confirming that it does not pause or wait for additional client-side input once initiated. Its primary role is to mutate the server's entity state via a `CommandBuffer`, ensuring that all changes are queued and processed safely within the main game loop.

## Lifecycle & Ownership
- **Creation:** Instances are created by the server's asset loading system during startup or asset reload. The static `CODEC` field is used to deserialize configuration data from game files into a fully-formed ModifyInventoryInteraction object. It is never instantiated directly with `new` in gameplay logic.
- **Scope:** An instance of this class is effectively a singleton for its specific configuration. It is an immutable template that persists for the entire server session. The same shared instance is used every time the corresponding interaction is triggered by any player.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server unloads its asset definitions, typically during a full server shutdown.

## Internal State & Concurrency
- **State:** **Immutable**. All fields that define the interaction's behavior (e.g., `itemToRemove`, `adjustHeldItemQuantity`) are set once during deserialization via the `CODEC`. The class itself holds no mutable state related to a specific gameplay event; all transactional state is passed in via the `InteractionContext` parameter in the `firstRun` method.
- **Thread Safety:** **Not thread-safe**. This class is designed to be executed exclusively by the server's main game loop thread. All state modifications are marshaled through the `CommandBuffer` provided by the `InteractionContext`. Direct invocation from other threads will lead to race conditions and severe state corruption.

## API Surface
The public contract is primarily defined by methods overridden from its parent class, which are invoked by the server's interaction processing system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(N) | Executes the core inventory logic. Complexity is dependent on inventory search operations. Mutates game state via the context's CommandBuffer. Can fail by setting the context's state to Failed. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Signals to the engine that this is a server-only, instantaneous interaction. |
| generatePacket() | Interaction | O(1) | Creates a network packet to inform the client of the inventory change. |
| configurePacket(packet) | void | O(1) | Populates the network packet with data from this interaction's configuration. |

## Integration Patterns

### Standard Usage
This class is not used directly in code. Instead, it is defined declaratively in an asset file. The engine then triggers this interaction based on player actions.

Below is a conceptual example of how this interaction might be defined in a JSON asset:

```json
// in some_item.json
{
  "id": "hytale:magic_crystal",
  "onUse": {
    "interaction": {
      "type": "ModifyInventoryInteraction",
      "adjustHeldItemQuantity": -1,
      "itemToAdd": {
        "id": "hytale:magic_dust",
        "quantity": 4
      }
    }
  }
}
```

The game engine would then be responsible for parsing this configuration and invoking the corresponding `ModifyInventoryInteraction` instance when a player uses the `magic_crystal` item.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ModifyInventoryInteraction()`. The object's state is meant to be populated by the `CODEC` during asset loading. Manual instantiation will result in a misconfigured and non-functional object.
- **State Mutation:** Do not attempt to modify the fields of a shared `ModifyInventoryInteraction` instance at runtime. These objects are treated as immutable templates.
- **Asynchronous Execution:** Do not call the `firstRun` method from outside the server's main tick loop. All interactions with the `InteractionContext` and its `CommandBuffer` must be synchronous with the game state updates.

## Data Pipeline
The data flow for this interaction is entirely server-authoritative. The client is only notified of the final state change.

> Flow:
> Player Action (e.g., Right Click) -> Network Event -> Server Interaction System -> **ModifyInventoryInteraction.firstRun()** -> CommandBuffer Execution -> Inventory & Entity State Change -> **ModifyInventoryInteraction.generatePacket()** -> Network Packet to Client -> Client-side Inventory UI Update

