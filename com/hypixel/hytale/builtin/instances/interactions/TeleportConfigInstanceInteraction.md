---
description: Architectural reference for TeleportConfigInstanceInteraction
---

# TeleportConfigInstanceInteraction

**Package:** com.hypixel.hytale.builtin.instances.interactions
**Type:** Transient

## Definition
```java
// Signature
public class TeleportConfigInstanceInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts

The TeleportConfigInstanceInteraction is a server-authoritative interaction handler responsible for teleporting a player to a dynamic game instance. It acts as the logical bridge between a player's physical interaction with a world block and the server's high-level instancing and world management system, the InstancesPlugin.

This class is not invoked directly. Instead, it is configured declaratively within game assets and associated with a specific block type. When a player interacts with a block of this type, the server's interaction module dispatches the event to this class's `interactWithBlock` method.

Its primary architectural function is to orchestrate a complex, and potentially asynchronous, workflow:
1.  **Configuration Retrieval:** It reads its operational parameters from a `ConfigurableInstanceBlock` component attached to the interacted block entity. This component specifies the target instance's name and other metadata.
2.  **Asynchronous World Provisioning:** It determines if the target instance world already exists. If not, it submits an asynchronous request to the `InstancesPlugin` to create and spawn the new world. This prevents the main server thread from blocking on expensive world generation.
3.  **Player Teleportation:** It adds a `Teleport` or `PendingTeleport` component to the interacting player's entity. The `PendingTeleport` is used when the target world is still being created, ensuring the player is moved only once the instance is ready.
4.  **State Management:** It manages the lifecycle of the interaction, including calculating a return point for the player and optionally scheduling the removal of the trigger block after the teleport is complete.

This interaction is fundamentally server-driven, as indicated by its `getWaitForDataFrom` implementation returning `Server`. This means the client performs no simulation and waits for the server to dictate the outcome, which is essential for an operation as significant as inter-world travel.

### Lifecycle & Ownership
-   **Creation:** Instances of this class are not created programmatically using `new`. They are deserialized from asset configuration files by the engine's `CODEC` system during server startup or asset loading. A single, shared instance typically represents a specific *type* of teleport interaction.
-   **Scope:** The object instance is long-lived, persisting for the entire server session. It is effectively a stateless service object whose methods are invoked with context-specific data for each interaction event.
-   **Destruction:** The instance is garbage collected when the server shuts down or performs a full asset reload that removes the corresponding interaction configuration.

## Internal State & Concurrency
-   **State:** This class is designed to be **stateless**. All data required for an operation, such as the world, player entity, and target block, is provided as method arguments to `interactWithBlock`. It does not contain mutable instance fields that persist across calls.
-   **Thread Safety:** The primary entry point, `interactWithBlock`, is designed to be called exclusively from the main server thread (the world tick). It safely initiates asynchronous operations using `CompletableFuture`. Callbacks that modify world state (e.g., updating the `ConfigurableInstanceBlock` component) are correctly scheduled to execute back on the main server thread via the world's executor, preventing race conditions.

    **WARNING:** While the class manages its own concurrency, any systems interacting with the `CompletableFuture` it returns must handle it appropriately and avoid blocking the main server thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `Server`, signaling that this interaction is fully server-authoritative and has no client-side prediction. |
| interactWithBlock(...) | void | Complex | The primary entry point. Orchestrates the entire teleport flow, from reading block configuration to initiating an asynchronous world spawn and commanding the player teleport. |
| simulateInteractWithBlock(...) | void | O(1) | Empty implementation. Reinforces that this interaction logic is not simulated on the client. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is configured as part of a block's properties within the game's asset files. The system automatically invokes it when a player triggers the interaction.

A conceptual configuration might look like this:

```json
// In a block asset file
{
  "interaction": {
    "type": "TeleportConfigInstanceInteraction",
    // ... other interaction properties like cooldowns
  }
}
```

The behavior is then further defined by the data within the `ConfigurableInstanceBlock` component attached to a specific block in the world.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new TeleportConfigInstanceInteraction()`. The class relies on being loaded and managed by the engine's asset and codec system.
-   **Manual Invocation:** Do not call `interactWithBlock` directly. Doing so bypasses the server's interaction pipeline, which manages critical aspects like permissions, cooldowns, and context creation.
-   **Blocking on Futures:** The `interactWithBlock` method may trigger a world spawn that returns a `CompletableFuture`. Code that interacts with this future must not block the main server thread (e.g., by calling `future.get()`). Use asynchronous callbacks like `thenAcceptAsync`.

## Data Pipeline
The flow of data and control for a teleport interaction is a multi-stage process orchestrated by the server.

> Flow:
> Player Input Packet -> Server Network Layer -> Interaction System -> **TeleportConfigInstanceInteraction.interactWithBlock** -> Reads `ConfigurableInstanceBlock` Component -> `InstancesPlugin.spawnInstance` -> `CommandBuffer` writes `PendingTeleport` Component to Player -> Entity Component System executes teleport at end of tick.

