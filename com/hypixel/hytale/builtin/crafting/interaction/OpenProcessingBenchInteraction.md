---
description: Architectural reference for OpenProcessingBenchInteraction
---

# OpenProcessingBenchInteraction

**Package:** com.hypixel.hytale.builtin.crafting.interaction
**Type:** Transient Handler

## Definition
```java
// Signature
public class OpenProcessingBenchInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts

OpenProcessingBenchInteraction is a server-side **Interaction Handler** that implements the logic for opening a user interface for a processing-style crafting station, such as a furnace, alloy forge, or cooking station.

This class is a key component of Hytale's data-driven block behavior system. It is not intended to be instantiated or managed directly in code. Instead, instances are created by the server's asset loading system via the provided static **CODEC**. This design allows game designers to define new processing benches in configuration files and associate them with this interaction logic without requiring new Java code.

Architecturally, this class acts as a controller that bridges a player's world interaction with the server's UI and state management systems. Its primary responsibilities are:
1.  **State Validation:** Verifying that the interacted block has a valid and correctly initialized ProcessingBenchState.
2.  **Session Management:** Creating a server-side ProcessingBenchWindow to represent the player's unique view of the bench.
3.  **State Registration:** Registering the player's session with the block's central ProcessingBenchState, allowing multiple players to view the same bench.
4.  **Client Notification:** Instructing the player's client to open the corresponding bench user interface via the PageManager.
5.  **Lifecycle Hooks:** Registering cleanup logic for when the player closes the UI, ensuring the block's state is correctly updated.

The `simulateInteractWithBlock` method is intentionally empty, indicating this interaction is entirely server-authoritative. There is no client-side prediction for opening a bench UI.

## Lifecycle & Ownership

-   **Creation:** A single instance of this class is typically instantiated by the server's Codec and asset management system during server startup. It is deserialized from game data definitions (e.g., a block's JSON file) that reference this interaction type.
-   **Scope:** The handler instance is application-scoped. It is stored in a central registry and reused for every single interaction with any block type configured to use it. It does not hold state specific to any single interaction.
-   **Destruction:** The object is de-referenced and eligible for garbage collection only when the server shuts down or performs a full asset reload.

## Internal State & Concurrency

-   **State:** OpenProcessingBenchInteraction is **stateless and immutable**. It contains no instance fields and its behavior is determined entirely by the arguments passed to the `interactWithBlock` method. All mutable state, such as the inventory of the bench or the list of players currently viewing it, is stored externally in the **ProcessingBenchState** object associated with the block entity in the world.

-   **Thread Safety:** This class is **not thread-safe**. Its methods are designed to be executed exclusively on the main server thread for a given world. Invoking `interactWithBlock` from a worker thread will lead to race conditions, world corruption, and server instability, as it directly manipulates non-thread-safe world and entity data structures.

## API Surface

The public contract is defined by its parent, SimpleBlockInteraction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) | Executes the primary server-side logic for opening the bench UI. Validates state, creates a window, and notifies the client. |
| simulateInteractWithBlock(...) | void | O(1) | No-op. This interaction is not predicted on the client. |

## Integration Patterns

### Standard Usage

This class is not intended for direct invocation by developers. It is automatically invoked by the server's core interaction module when a player interacts with a block that has been configured to use this handler in its asset definition.

The following conceptual example shows how a game designer would associate this interaction with a block in a JSON asset file.

```json
// Conceptual asset file: "hytale:alloy_forge.json"
{
  "block": {
    "id": "hytale:alloy_forge",
    "blockState": "ProcessingBenchState",
    "interactions": [
      {
        "type": "OpenProcessingBenchInteraction",
        "interactionType": "RIGHT_CLICK"
      }
    ]
  }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new OpenProcessingBenchInteraction()`. The interaction system relies on the static CODEC for deserialization and registration. Manually created instances will not be recognized by the engine.
-   **Incorrect State Association:** Do not configure a block to use this interaction unless its BlockState is, or inherits from, ProcessingBenchState. The handler will fail at runtime, sending an error message to the player and potentially resetting the block's state.
-   **Manual Invocation:** Never call the `interactWithBlock` method directly. It must be dispatched by the server's interaction loop, which provides a valid and synchronized CommandBuffer, InteractionContext, and other required parameters.

## Data Pipeline

The flow of data and control for this interaction spans from client input to a client UI update, with this class acting as the central server-side controller.

> Flow:
> Player Input (Right-Click) -> Client sends `InteractPacket` -> Server Network Layer -> Server Interaction Module dispatches to **OpenProcessingBenchInteraction** -> Handler validates `ProcessingBenchState` -> Handler creates `ProcessingBenchWindow` -> Handler calls `PageManager.setPageWithWindows` -> Server sends `OpenPagePacket` -> Client UI System receives packet and renders the bench interface.

