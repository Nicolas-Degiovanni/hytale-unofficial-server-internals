---
description: Architectural reference for UseBlockInteraction
---

# UseBlockInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient Behavioral Object

## Definition
```java
// Signature
public class UseBlockInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts

The UseBlockInteraction class is a concrete implementation of the **Strategy Pattern** within the server's core interaction system. It represents the specific, high-level action of an entity "using" a block, typically triggered by a right-click input.

This class does not contain the ultimate logic for what happens when a block is used. Instead, it functions as a **configurable dispatcher**. Its primary responsibility is to:
1.  Identify the target block in the world.
2.  Query the block's `BlockType` configuration data.
3.  Look up the name of a secondary interaction to execute based on the `InteractionType` (e.g., PRIMARY, SECONDARY).
4.  Trigger the server's event bus with cancellable pre-interaction events.
5.  Delegate the final execution to the configured secondary interaction, such as running a script or opening a container.

This design decouples the low-level player action from the high-level, data-driven game logic defined in block assets. The clear separation between `interactWithBlock` (full execution with events) and `simulateInteractWithBlock` (logic-only) is a critical feature to support server-authoritative simulation while allowing for client-side prediction without producing duplicate side-effects.

## Lifecycle & Ownership

-   **Creation:** Instances of UseBlockInteraction are not instantiated directly via a constructor in game logic. They are deserialized from server asset configuration files at startup by the engine's `BuilderCodec` system. Each instance represents a reusable, stateless definition of the "use block" behavior.
-   **Scope:** As a configured and stateless object, a single shared instance of UseBlockInteraction persists for the entire server session, managed by a central interaction registry.
-   **Destruction:** The instance is destroyed when the server shuts down and its asset registries are cleared from memory.

## Internal State & Concurrency

-   **State:** UseBlockInteraction is **stateless and immutable**. It contains no instance fields and its behavior is determined entirely by the arguments passed to its methods, such as the `InteractionContext` and `World`. All relevant state is managed externally.

-   **Thread Safety:** The class is inherently **thread-safe**. However, it operates on systems like `World` and `EntityStore` which are not. All mutations to the game state are funneled through the provided `CommandBuffer`. This mechanism ensures that all state changes are queued and executed serially on the main game thread, preventing race conditions and maintaining world integrity.

    **Warning:** Any direct modification of the `World` or `EntityStore` from within this class would violate the engine's concurrency model and lead to severe data corruption.

## API Surface

The public API is minimal, designed for invocation by the server's central interaction handler, not by general game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) | Executes the full interaction, including firing pre/post events. The complexity is for the dispatch itself; the delegated interaction may be more complex. |
| simulateInteractWithBlock(...) | void | O(1) | Executes the interaction logic without firing events. Used for server-side validation or client-side prediction. |
| generatePacket() | Interaction | O(1) | Constructs the corresponding network protocol packet for this interaction type. |

## Integration Patterns

### Standard Usage

This class is not intended for direct developer use. It is invoked by the server's core interaction module after a corresponding network packet is received and decoded. The system retrieves this configured object from a registry and executes it.

```java
// Hypothetical invocation from a central interaction handler
InteractionContext context = createInteractionContextForPlayer(player);
Vector3i targetBlock = getPlayerLookTarget(player);
InteractionType type = InteractionType.PRIMARY;

// The handler would look up the correct interaction object (e.g., UseBlockInteraction)
// and then dispatch to it.
// This is a simplified representation.
configuredInteraction.interactWithBlock(
    world,
    context.getCommandBuffer(),
    type,
    context,
    player.getHeldItemStack(),
    targetBlock,
    player.getCooldownHandler()
);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new UseBlockInteraction()`. The class is designed to be configured and loaded from assets. Direct creation bypasses this system and will result in a non-functional object.
-   **Stateful Logic:** Do not extend this class to add state. Its statelessness is fundamental to its role as a reusable, thread-safe dispatcher.
-   **Bypassing the Command Buffer:** Never modify the `World` or entities directly. All state changes must be enqueued on the `CommandBuffer` provided in the `InteractionContext` to ensure thread safety.

## Data Pipeline

The flow of data and control for a typical "use block" action is sequential and passes through several key systems.

> Flow:
> Player Input -> Client Network Layer -> `UseBlockInteraction` Packet -> Server Network Layer -> Interaction Module -> **UseBlockInteraction.interactWithBlock** -> `BlockType` Config Lookup -> `UseBlockEvent.Pre` (Event Bus) -> Delegated `RootInteraction` Execution -> `UseBlockEvent.Post` (Event Bus) -> Queued `CommandBuffer` Execution -> Final Game State Change

