---
description: Architectural reference for DestroyBlockInteraction
---

# DestroyBlockInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class DestroyBlockInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The DestroyBlockInteraction class is a server-side, data-driven implementation of a discrete game action. It represents the specific logic required to destroy a single block within the world. As a subclass of SimpleInstantInteraction, it is designed to execute completely within a single server tick without maintaining any state across multiple ticks.

Its primary architectural significance lies in its integration with the Hytale `CODEC` system. The static CODEC field allows instances of this class to be defined entirely within external configuration files (e.g., JSON or HOCON). This decouples the low-level game logic from the high-level game design, enabling designers to create or modify block-breaking behaviors without recompiling the server source code.

This class acts as a command object within the server's Interaction Module. When an entity's action is resolved to this specific interaction, the `firstRun` method is invoked with an InteractionContext, which provides a complete snapshot of the required world state for the operation. It then uses a CommandBuffer to queue world modifications, ensuring that changes are applied deterministically at the end of the current game tick.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly using the `new` keyword. They are instantiated by the engine's serialization system when parsing game asset configuration files. The `CODEC` field is the factory responsible for this deserialization.
-   **Scope:** An instance of DestroyBlockInteraction, once loaded from configuration, persists for the entire server session. It is held in a central registry of available interactions and is reused for every corresponding block destruction event.
-   **Destruction:** The object is garbage collected when the server shuts down or when a hot-reload of game assets purges the old interaction registry. Its lifecycle is managed entirely by the asset loading and interaction systems.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. It contains no member fields for storing instance-specific data. All necessary information, such as the target block and the acting entity, is provided via the InteractionContext parameter in the `firstRun` method. This design ensures that a single shared instance can be used for all block destruction events without side effects.
-   **Thread Safety:** This class is **not thread-safe** and must only be used from the main server thread that manages the corresponding world. It directly accesses and manipulates world state components like ChunkStore and uses a CommandBuffer, which are not designed for concurrent access. Invoking its methods from an external thread will lead to world state corruption, race conditions, and server instability.

## API Surface
The public contract is defined by its parent class, SimpleInstantInteraction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Executes the block destruction logic. Validates the target block from the context, locates the relevant chunk, and delegates the core operation to BlockHarvestUtils. Throws NullPointerException if the context or its internal data is invalid. |

## Integration Patterns

### Standard Usage
Developers do not typically interact with this class directly in Java code. Its primary use is through asset configuration files. The system invokes it automatically.

The following conceptual example shows how the server's Interaction Module would execute this action after resolving it from a player input event.

```java
// System-level code that invokes the interaction
InteractionContext context = createInteractionContextForPlayer(player, target);
Interaction resolvedInteraction = interactionRegistry.get("hytale:destroy_block");

// The system calls the method on the pre-loaded instance
if (resolvedInteraction instanceof SimpleInstantInteraction) {
    ((SimpleInstantInteraction) resolvedInteraction).firstRun(InteractionType.PRIMARY, context, player.getCooldownHandler());
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new DestroyBlockInteraction()`. This bypasses the configuration system and creates an unmanaged object. All interactions must be defined in and loaded from asset files.
-   **Asynchronous Execution:** Do not call `firstRun` from a separate thread or an asynchronous task. All world modifications must be synchronized with the main server tick to prevent state corruption.
-   **Stateful Subclassing:** Do not extend this class to add stateful member variables. The design relies on a shared, stateless instance.

## Data Pipeline
The flow of data and control for a block destruction event is a multi-stage process that spans from client input to server-side world mutation.

> Flow:
> Client Input (Mouse Click) -> Network Packet -> Server Protocol Layer -> Interaction Module -> **DestroyBlockInteraction.firstRun()** -> BlockHarvestUtils -> CommandBuffer -> World State Update -> Network Replication to Clients

