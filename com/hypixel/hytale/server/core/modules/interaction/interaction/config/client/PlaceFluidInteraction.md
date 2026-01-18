---
description: Architectural reference for PlaceFluidInteraction
---

# PlaceFluidInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient

## Definition
```java
// Signature
public class PlaceFluidInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The PlaceFluidInteraction is a server-side, data-driven command that defines the logic for placing a fluid in the world. It is a concrete implementation within the server's Interaction System, which processes player actions like clicking a mouse button.

This class extends SimpleBlockInteraction, inheriting the foundational behavior for interactions that target a specific block. Its primary role is to translate a player's intent to place a fluid (e.g., from a bucket) into a direct modification of the world's data structures.

Crucially, PlaceFluidInteraction is designed to be configured from external asset files via its static CODEC field. This allows game designers to define new items or behaviors that place specific fluids without changing Java code. The class acts as the runtime executor for this declarative configuration. The method `getWaitForDataFrom` returning `Client` signifies that this interaction is initiated by a client-side action, and the server executes it authoritatively.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly using the `new` keyword in game logic. They are deserialized from asset configuration files by the engine's `BuilderCodec` system during server startup. These configured instances are then registered within the Interaction Module.
-   **Scope:** A configured instance of PlaceFluidInteraction persists for the entire server session. It acts as a stateless template for an action that can be executed repeatedly.
-   **Destruction:** Instances are garbage collected when the server shuts down and all game assets are unloaded.

## Internal State & Concurrency
-   **State:** The internal state consists of `fluidKey` and `removeItemInHand`. This state is populated once during asset loading and is considered immutable for the remainder of the server session. The class itself is stateless with respect to its execution; all required context for an operation is passed as arguments to the `interactWithBlock` method.
-   **Thread Safety:** The class is thread-safe for reads. The `interactWithBlock` method, which mutates world state, is designed to be executed within the main server thread or a world-specific update tick. It leverages a `CommandBuffer` to queue entity state changes, a standard pattern to defer mutations and prevent race conditions. Direct invocation from an uncontrolled, asynchronous thread will lead to world state corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) | Executes the fluid placement logic. Modifies the target `FluidSection` and updates the chunk's ticking state. Throws exceptions if world data is in an inconsistent state. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `Client`, indicating the server should wait for client input to trigger this interaction. |
| needsRemoteSync() | boolean | O(1) | Returns `true`, signaling to the engine that the world change must be synchronized with all relevant clients. |
| simulateInteractWithBlock(...) | void | O(1) | An empty implementation. This interaction has no client-side prediction logic; it is fully server-authoritative. |

## Integration Patterns

### Standard Usage
A developer does not invoke this class directly. It is triggered by the server's core Interaction Module in response to a client network packet. The system identifies the active interaction and executes it.

```java
// Hypothetical usage within the server's InteractionModule

// 1. An interaction is identified based on player state and input
PlaceFluidInteraction interaction = getInteractionForItem(player.getHeldItem());

// 2. The system provides the necessary context and executes the interaction
if (interaction != null) {
    interaction.interactWithBlock(
        world,
        commandBuffer,
        interactionType,
        interactionContext,
        itemInHand,
        targetBlock,
        cooldownHandler
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PlaceFluidInteraction()`. This bypasses the data-driven configuration system and will result in an unconfigured object that cannot function correctly.
-   **Runtime State Mutation:** Do not attempt to modify the `fluidKey` or other fields after the server has loaded. These are considered static configuration.
-   **External Invocation:** Do not call `interactWithBlock` from outside the server's main update loop or without the correct `CommandBuffer` and `World` context. This will bypass thread safety mechanisms and corrupt game state.

## Data Pipeline
The class is a key processing step in the player interaction data pipeline. It translates a high-level player action into a low-level, concrete world mutation.

> Flow:
> Client Input (Click) -> Network Packet -> Server Interaction Module -> **PlaceFluidInteraction.interactWithBlock** -> World ChunkStore & CommandBuffer Mutation -> Network Sync System -> Client World Render Update

