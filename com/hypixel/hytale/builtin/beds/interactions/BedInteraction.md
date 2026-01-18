---
description: Architectural reference for BedInteraction
---

# BedInteraction

**Package:** com.hypixel.hytale.builtin.beds.interactions
**Type:** Transient Handler

## Definition
```java
// Signature
public class BedInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The BedInteraction class is a server-side gameplay logic handler responsible for all player interactions with bed blocks. It is a concrete implementation of the engine's generic block interaction system, specializing it for the complex state machine involved in sleeping and managing respawn points.

This class acts as a central controller that is invoked by the core Interaction Module when a player targets a bed. Its primary function is to analyze the context of the interaction—who the player is, who owns the bed, and the player's existing respawn configuration—and then dispatch the appropriate commands. It orchestrates a workflow that can result in one of several outcomes:

1.  **Mounting & Sleeping:** If the player owns the bed, it initiates the sleeping process by mounting the player entity onto the block and adding the PlayerSomnolence component.
2.  **Respawn Point UI:** If the bed is unowned, it triggers a multi-page UI flow for the player to claim the bed as a new respawn point. This involves checking against existing respawn points and server-defined limits.
3.  **Feedback Message:** If the bed is owned by another player, it sends a simple notification message to the interacting player.

It operates exclusively on the server and uses a CommandBuffer to ensure that all state mutations are queued and executed deterministically at the end of the current server tick.

### Lifecycle & Ownership
-   **Creation:** An instance of BedInteraction is not created per interaction. Instead, it is instantiated once at server startup by the asset loading system. The static CODEC field is used to deserialize this class from a gameplay configuration file, effectively binding this logic to specific block types.
-   **Scope:** The singleton instance created from configuration persists for the entire server session. It is held in a central registry managed by the Interaction Module.
-   **Destruction:** The instance is destroyed during server shutdown when all asset registries are cleared from memory.

## Internal State & Concurrency
-   **State:** BedInteraction is **completely stateless**. It contains no mutable member fields and all data required for its operations is provided as arguments to the interactWithBlock method. This design makes the class inherently reusable and predictable.

-   **Thread Safety:** This class is designed to be called exclusively from the main server thread during the world tick processing phase. It is not thread-safe for concurrent invocations. All world state modifications are funneled through the provided CommandBuffer, which is a critical engine pattern for maintaining thread safety and data consistency. The CommandBuffer defers all entity component changes until a safe point in the tick cycle, preventing race conditions.

## API Surface
The public contract is minimal, with a single method driving all behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(N) | The primary entry point. Orchestrates the entire bed interaction logic. Complexity is O(N) where N is the number of the player's saved respawn points, due to the proximity check. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by gameplay developers. It is automatically invoked by the server's Interaction Module. A developer would typically configure a block asset to use this handler.

The following example illustrates how the *engine* would use this class.

```java
// Engine-level code (conceptual)
// A configured instance is retrieved from a registry
BedInteraction handler = interactionRegistry.getHandlerForBlock(bedBlock);

// The engine invokes the handler with the current interaction context
handler.interactWithBlock(world, commandBuffer, type, context, item, pos, cooldowns);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new BedInteraction()`. The behavior of this class is tied to its deserialization via the static CODEC. Bypassing this mechanism will lead to an unconfigured and non-functional object.
-   **Direct World Modification:** Do not attempt to get components from the world and modify them directly within this method. All state changes **must** be queued through the provided CommandBuffer instance to prevent state corruption and ensure determinism.
-   **Asynchronous Operations:** Do not perform blocking I/O or launch asynchronous tasks from within interactWithBlock. It is expected to execute synchronously and quickly within a single server tick.

## Data Pipeline
The flow of data for a bed interaction is unidirectional, originating from a client packet and resulting in a server-side state change or a new UI page sent back to the client.

> Flow:
> Client Input Packet -> Server Network Module -> Interaction Module -> **BedInteraction.interactWithBlock** -> CommandBuffer (for state changes) OR PageManager (for UI) -> State Update / Client UI Packet

