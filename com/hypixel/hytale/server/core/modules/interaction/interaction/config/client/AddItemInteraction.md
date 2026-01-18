---
description: Architectural reference for AddItemInteraction
---

# AddItemInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Configuration Model

## Definition
```java
// Signature
public class AddItemInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The AddItemInteraction class is a data-driven, behavioral component within the server's Interaction System. It is not a long-lived service or manager; rather, it represents a single, configurable action: adding a specified item to a player's inventory when they interact with a block.

Its primary architectural feature is the static CODEC field. This enables the Hytale engine to deserialize AddItemInteraction instances directly from game asset files (e.g., JSON definitions for blocks). This pattern allows designers and developers to define complex game logic through simple data configuration without writing new Java code for each variant.

This class extends SimpleBlockInteraction, inheriting the contract for server-side block interaction logic. Its responsibility is narrowly focused on mutating the interacting entity's inventory state via a CommandBuffer, ensuring that the operation is correctly queued and processed within the server's main tick loop. The empty implementation of simulateInteractWithBlock signifies that this is a server-authoritative action with no client-side prediction.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly using the new keyword. They are instantiated by the engine's Codec system during server startup when game assets are loaded. The BuilderCodec reads properties like ItemId and Quantity from a configuration file and uses them to construct the object.

- **Scope:** An instance of AddItemInteraction persists for the lifetime of the configuration that defines it. It is typically owned by a higher-level configuration object, such as a block definition, and is held in a server-wide registry. The object itself is stateless after creation, allowing a single instance to be reused for all interactions of its type.

- **Destruction:** The object is marked for garbage collection when its owning configuration is unloaded, typically during an asset reload or server shutdown.

## Internal State & Concurrency
- **State:** The internal state, consisting of itemId and quantity, is **effectively immutable**. These values are set once during deserialization from a configuration file and are not designed to be modified at runtime.

- **Thread Safety:** The object is inherently thread-safe due to its immutable state. However, the methods it implements, such as interactWithBlock, are **not safe to call from arbitrary threads**. These methods operate on sensitive, mutable game state like the World and EntityStore. The engine's architecture guarantees that these interaction handlers are only ever invoked from the main server thread, which has exclusive write access to the game world, thus preventing race conditions.

## API Surface
The public contract is defined by its parent class, SimpleBlockInteraction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) | Executes the item-granting logic. Retrieves the interacting entity, validates it is a LivingEntity, and queues a command to add the configured ItemStack to its inventory. This operation is transactional through the CommandBuffer. |
| simulateInteractWithBlock(...) | void | O(1) | No-op. This interaction has no client-side simulation component. |

## Integration Patterns

### Standard Usage
This class is not used directly in procedural code. Instead, it is declared within a data file, such as a block's JSON definition. The server's Interaction Module will automatically load this configuration and trigger it when appropriate.

A developer would define this behavior in an asset file like so:
```json
// Example: my_reward_block.json
{
  "id": "my_reward_block",
  "interactions": [
    {
      "type": "AddItemInteraction",
      "ItemId": "hytale:gold_coin",
      "Quantity": 10
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new AddItemInteraction()`. This bypasses the entire data-driven design of the engine. All interactions must be defined in asset files to be loaded by the server's Codec system.

- **Runtime State Mutation:** Do not attempt to modify the itemId or quantity fields after the object has been created. These are considered static configuration and changing them at runtime can lead to unpredictable behavior.

- **External Invocation:** Never call the interactWithBlock method from your own code. It is designed to be invoked exclusively by the server's core Interaction Module, which provides the correct context, state, and threading guarantees.

## Data Pipeline
The flow of data and execution for this component begins with configuration and ends with a change in the game state.

> Flow:
> Game Asset File (JSON) -> Server Asset Loader -> **BuilderCodec** -> **AddItemInteraction Instance** (In Memory) -> Player Interaction Event -> Interaction Module -> **interactWithBlock()** -> CommandBuffer -> Entity Inventory Update

