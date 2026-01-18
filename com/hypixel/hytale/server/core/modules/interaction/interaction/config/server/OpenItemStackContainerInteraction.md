---
description: Architectural reference for OpenItemStackContainerInteraction
---

# OpenItemStackContainerInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Transient / Data-Driven

## Definition
```java
// Signature
public class OpenItemStackContainerInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The OpenItemStackContainerInteraction is a specific, stateless implementation of the server-side interaction system. It represents a single, instantaneous action that can be configured and attached to in-game items. Its primary architectural role is to serve as a bridge between a player's world interaction (e.g., right-clicking while holding an item) and the server's User Interface and Inventory systems.

This class is designed to be data-driven. It is not intended to be instantiated or managed directly in procedural code. Instead, game designers define its usage in asset configuration files. The engine deserializes these configurations at runtime using the provided BuilderCodec, linking this specific logic to an item.

When triggered, its core responsibility is to:
1.  Validate the interaction context, ensuring a player is performing the action and is not already engaged in another UI.
2.  Inspect the item stack the player is holding.
3.  Read the item's `ItemStackContainerConfig` to understand the properties of the container it holds.
4.  Ensure a persistent `ItemStackItemContainer` exists for that specific item instance.
5.  Instruct the player's `PageManager` to open a new UI page (`Page.Bench`) and display the item's container contents using an `ItemStackContainerWindow`.

This component operates exclusively on the server and communicates UI changes to the client via the `PageManager` and the underlying network protocol.

### Lifecycle & Ownership
-   **Creation:** Instances are deserialized from game data files (e.g., JSON) by the server's asset loading pipeline at startup. The public static `CODEC` field is the entry point for this deserialization process. Direct instantiation via `new` is a design violation.
-   **Scope:** An instance of this class is effectively a stateless template. A single, shared instance is held in a central registry and reused for every interaction of this type. Its lifetime is coupled to the server session.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down or the asset configurations are reloaded. No explicit destruction logic is required.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. It contains no member fields to store per-interaction data. All required state is provided via the `InteractionContext` parameter in the `firstRun` method, making the object itself inherently reusable and predictable.

-   **Thread Safety:** The class is thread-safe by design due to its statelessness. However, the execution of its methods is **not** guaranteed to be safe if invoked from multiple threads simultaneously. The `firstRun` method operates on the Entity-Component-System (ECS) state and must be executed on the main server game thread. It correctly uses a `CommandBuffer` to queue state mutations, a pattern which ensures that all changes are applied atomically at a safe point in the server tick, preventing race conditions and data corruption within the ECS.

    **Warning:** Any modifications to this class that introduce mutable state must be protected by appropriate synchronization mechanisms, though doing so would violate its core design principles.

## API Surface
The public API is minimal, consisting of the framework-level codec and the overridden logic method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Executes the interaction logic. Retrieves components from the ECS and queues a command to open a UI page. This is the primary entry point, called by the interaction system. |

## Integration Patterns

### Standard Usage
This class is not used directly in code. Instead, it is specified as the interaction handler within an item's asset definition file. The engine's interaction module resolves and executes this logic automatically.

A conceptual item configuration might look like this:
```json
// Example: my_item.json
{
  "id": "hytale:magic_pouch",
  "interaction": {
    "type": "OpenItemStackContainerInteraction"
  },
  "components": {
    "itemStackContainerConfig": {
      "size": 9,
      "title": "Magic Pouch"
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new OpenItemStackContainerInteraction()`. The interaction system relies on the centrally managed, deserialized instances. Bypassing this will lead to unhandled and untracked behavior.
-   **Manual Invocation:** Do not call the `firstRun` method directly. The `InteractionContext` is a complex, stateful object constructed deep within the server's interaction pipeline. Attempting to mock or manually create it will result in `NullPointerException` or inconsistent game state.
-   **Direct State Mutation:** Do not modify ECS components directly from within `firstRun`. Always use the provided `CommandBuffer` to ensure changes are applied safely within the server's tick lifecycle.

## Data Pipeline
The flow of data and control for this interaction begins with player input and ends with a UI update on the client.

> Flow:
> Player Input (Right-Click) -> Client Network Packet -> Server Interaction System -> Resolves Item Interaction -> **OpenItemStackContainerInteraction.firstRun()** -> CommandBuffer.setComponent(PageManager) -> End-of-Tick System Flush -> Server Network Packet (UI Open) -> Client UI System
---

