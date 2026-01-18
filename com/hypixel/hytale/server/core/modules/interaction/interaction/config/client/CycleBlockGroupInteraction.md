---
description: Architectural reference for CycleBlockGroupInteraction
---

# CycleBlockGroupInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient / Configuration

## Definition
```java
// Signature
public class CycleBlockGroupInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The CycleBlockGroupInteraction is a server-side behavior definition that implements a specific game mechanic: cycling a block through a predefined sequence of related blocks. It is a concrete implementation within the server's broader Interaction System, which processes player and entity actions upon the game world.

This class acts as a data-driven command. It is not instantiated directly in code but is deserialized from asset files via its static CODEC. This allows game designers to configure which items or tools trigger this "cycling" behavior on which types of blocks without requiring engine code changes.

Its primary architectural role is to translate a high-level interaction event into low-level world state mutations. It achieves this by:
1.  **Querying World State:** Reading the current block type at a specific coordinate from the world's chunk data.
2.  **Consulting Asset Data:** Cross-referencing the block with its corresponding BlockGroup asset to determine the next block in the sequence.
3.  **Mutating World State:** Enqueuing a command to replace the existing block with the new one.
4.  **Triggering Side Effects:** Causing item durability loss and playing sound effects.

Crucially, all world mutations are deferred through a CommandBuffer. This is a standard pattern in the engine to ensure that state changes are applied atomically and in a deterministic order at the end of a game tick, preventing race conditions and maintaining world integrity.

## Lifecycle & Ownership
-   **Creation:** Instances are not created manually using the new keyword. They are instantiated by the server's asset loading system during bootstrap or asset hot-reloading. The static CODEC field is the entry point for this deserialization process.
-   **Scope:** An instance of this class is effectively a stateless singleton tied to a specific interaction configuration. It persists for the entire server session, or until assets are reloaded. It represents the *definition* of the interaction, not a single occurrence of it.
-   **Destruction:** Instances are garbage collected when the server's asset registry is cleared, typically during a server shutdown or a full asset reload.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. It contains no instance fields and all necessary data is provided as arguments to its methods. This design makes it inherently reusable and predictable for any number of simultaneous interactions of this type.

-   **Thread Safety:** The class is conditionally thread-safe. Because it is stateless, its methods can be safely invoked from any thread. However, the engine guarantees safety by processing world interactions within a single, dedicated thread per world or by using a job system that leverages the CommandBuffer pattern. Direct, unmanaged concurrent calls to `interactWithBlock` with the same World object would be unsafe. The engine's architecture prevents this.

## API Surface
The public contract is focused on executing the interaction logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) | Executes the block cycling logic. Throws assertions if world components are missing. All state is passed via arguments. |
| simulateInteractWithBlock(...) | void | O(1) | A no-op in this implementation. Intended for client-side prediction, which is not relevant for this server-side action. |
| CODEC | BuilderCodec | N/A | Static field used by the asset system to deserialize this class from configuration files. |

## Integration Patterns

### Standard Usage
A developer does not invoke this class directly. The server's InteractionModule is responsible for resolving an interaction event to a configured behavior like this one and dispatching the call.

```java
// Hypothetical engine code in an InteractionModule
InteractionContext context = ...; // Build context from network packet
SimpleBlockInteraction interactionLogic = context.getInteractionLogic(); // Resolved from assets

// The engine dispatches the call. The resolved logic could be a CycleBlockGroupInteraction instance.
if (interactionLogic != null) {
    interactionLogic.interactWithBlock(world, commandBuffer, type, context, ...);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new CycleBlockGroupInteraction()`. The behavior is data-driven and must be loaded from game assets. Manual creation bypasses the configuration system.
-   **Stateful Implementation:** Adding mutable instance fields to this class would violate its core design. State should be managed in components and passed via the InteractionContext, not stored on the interaction definition itself.
-   **Bypassing the CommandBuffer:** Directly modifying a WorldChunk or BlockChunk component from within this class would break the engine's deterministic, end-of-tick state update model and could lead to severe concurrency bugs and world corruption.

## Data Pipeline
The flow of data through this component is linear and transforms an interaction request into a series of deferred commands.

> Flow:
> Player Interaction Packet -> InteractionModule -> **CycleBlockGroupInteraction.interactWithBlock**
> 1.  Read `targetBlock` from `InteractionContext`.
> 2.  Fetch `WorldChunk` and `BlockChunk` from `World`.
> 3.  Read current block ID from `BlockSection`.
> 4.  Look up `BlockType` asset from block ID.
> 5.  Look up `BlockGroup` asset from `BlockType`.
> 6.  Calculate next `BlockType` in the group.
> 7.  Queue `WorldChunk.setBlock` in `CommandBuffer`.
> 8.  Queue `SoundUtil.playSoundEvent3d` in `CommandBuffer`.
> 9.  Update `InteractionState` on `InteractionContext`.

