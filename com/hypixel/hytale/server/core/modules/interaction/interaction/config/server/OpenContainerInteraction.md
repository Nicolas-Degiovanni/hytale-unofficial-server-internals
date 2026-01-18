---
description: Architectural reference for OpenContainerInteraction
---

# OpenContainerInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Behavior Definition / Transient

## Definition
```java
// Signature
public class OpenContainerInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The **OpenContainerInteraction** class is a server-side, data-driven behavior definition that governs how players open user interface windows for blocks that function as containers, such as chests or furnaces. It is not a persistent service but rather a concrete implementation of the interaction contract defined by **SimpleBlockInteraction**.

This class acts as the central logic hub for the entire "open container" workflow. Its primary responsibilities include:
1.  **State Validation:** Verifying that the target block possesses a valid **ItemContainerState**.
2.  **Permission Checks:** Ensuring the interacting player has the rights to view the container's contents.
3.  **Session Management:** Tracking which players currently have a specific container's UI open. It orchestrates the creation and destruction of server-side **ContainerBlockWindow** objects, which represent a player's view into a container.
4.  **State Transition:** Modifying the block's world state to trigger visual or behavioral changes, such as animating a chest lid opening or closing. This is achieved by setting interaction states like "OpenWindow" and "CloseWindow".
5.  **Client Synchronization:** Commanding the player's client to open the appropriate UI page via the **Player.PageManager**.
6.  **Auditory Feedback:** Triggering sound effects that correspond to the opening and closing of the container.

Architecturally, it is a leaf node in the interaction system, invoked by higher-level modules when a player's input is resolved to this specific behavior.

### Lifecycle & Ownership
-   **Creation:** **OpenContainerInteraction** is instantiated at server startup by the asset loading system. Its static **CODEC** field dictates how it is deserialized from game data files. It is never instantiated directly with `new` during gameplay. A single, shared instance of this class defines the behavior for all block types that reference it.
-   **Scope:** The object instance is application-scoped, persisting for the entire server session. However, its execution via the **interactWithBlock** method is transaction-scoped, lasting only for the duration of a single interaction event within a single game tick.
-   **Destruction:** The instance is destroyed when the server shuts down or performs a full asset reload.

## Internal State & Concurrency
-   **State:** This class is **immutable and stateless**. It contains no member variables that are modified during gameplay. All state it operates on, such as the **World**, the target **BlockState**, and the interacting player's components, is passed in as method parameters.
-   **Thread Safety:** The class itself is inherently thread-safe due to its stateless design. However, its methods are **not re-entrant** and are fundamentally unsafe to call from any thread other than the main world-update thread. The methods manipulate core game state components like **World** and **CommandBuffer**, which are not designed for concurrent access. The engine's single-threaded, command-buffer-based architecture must be strictly respected.

## API Surface
The primary contract is fulfilled by overriding methods from its parent, **SimpleBlockInteraction**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | protected void | O(log N) | Core logic execution. Validates state, manages UI windows, and triggers block state changes. Complexity is dominated by map operations on the container's viewer list (N = number of current viewers). |
| simulateInteractWithBlock(...) | protected void | O(1) | No-op. This interaction is purely server-authoritative and has no client-side prediction component. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly from gameplay code. Instead, it is specified within a block's asset definition files. The server's Interaction Module resolves a player's action to the configured interaction type and executes it automatically.

The conceptual flow within the engine is as follows:

```java
// Engine-level code (conceptual)
// A player interacts with a block at 'pos'
InteractionDefinition interaction = world.getBlockType(pos).getInteraction();

// The engine identifies this as an OpenContainerInteraction
if (interaction instanceof OpenContainerInteraction containerInteraction) {
    // The engine invokes the behavior
    containerInteraction.interactWithBlock(world, commandBuffer, ...);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new OpenContainerInteraction()`. The behavior must be loaded via the asset system's **CODEC** to be correctly integrated into the engine.
-   **Incorrect Configuration:** Assigning this interaction to a block asset that does not use an **ItemContainerState** in its world representation. This will lead to failed interactions and error messages being sent to the player.
-   **Asynchronous Execution:** Calling **interactWithBlock** from a separate thread. This will cause severe state corruption, race conditions, and server instability. All interaction logic must be executed on the main server tick.

## Data Pipeline
The flow of data and control for a successful container interaction is a round-trip process, beginning with player input and ending with a UI update on their client.

> **Control Flow:**
> Client Input (Right Click) → Network Packet → Server Network Layer → Interaction Module → **OpenContainerInteraction.interactWithBlock** → CommandBuffer → Player.PageManager → Network Packet → Client UI Update

> **State Mutation Flow:**
> **OpenContainerInteraction** → World.getState(pos) → ItemContainerState.getWindows() → Map.putIfAbsent(playerUUID, window) → World.setBlockInteractionState(pos, "OpenWindow") → SoundUtil.playSoundEvent3d(...)

