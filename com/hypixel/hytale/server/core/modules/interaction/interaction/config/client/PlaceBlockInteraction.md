---
description: Architectural reference for PlaceBlockInteraction
---

# PlaceBlockInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Configuration / Data Object

## Definition
```java
// Signature
public class PlaceBlockInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The PlaceBlockInteraction class is a server-side, data-driven implementation defining the logic for placing a block in the world. It is a concrete specialization within the server's Interaction System, which is responsible for processing and executing player actions.

Architecturally, this class serves as a stateless processor for a specific game action. Its behavior is not hardcoded but is defined and configured through its static CODEC. This allows game designers to create many variations of block placement behavior (e.g., placing a specific block regardless of the held item, consuming or not consuming the item) purely through data files, without changing engine code.

A critical design aspect is its data dependency, specified by `getWaitForDataFrom` returning `WaitForDataFrom.Client`. This establishes a client-authoritative interaction model for block placement. The server does not attempt to predict where the player will place a block; instead, it waits for the client to send precise `InteractionSyncData` containing the target block position and rotation. This model prioritizes client-side responsiveness, as the client can render the placement preview immediately, while the server acts as a final validator and executor of the action.

### Lifecycle & Ownership
-   **Creation:** Instances are not created manually. They are deserialized and instantiated by the engine's `CODEC` system during server startup or when game assets are loaded. Each unique block placement interaction defined in the game's content files will correspond to a single, cached instance of this class.
-   **Scope:** Application-scoped. An instance persists for the entire server session, acting as a shared, immutable template for its corresponding interaction type.
-   **Destruction:** Instances are garbage collected when the server shuts down or performs a full asset reload.

## Internal State & Concurrency
-   **State:** This class is effectively immutable after its initial creation by the CODEC. Its fields, such as `blockTypeKey` and `removeItemInHand`, are configured once during asset loading and are never modified during gameplay. All mutable state related to a specific interaction *event* is passed in via the `InteractionContext` parameter in the `tick0` method.
-   **Thread Safety:** The class is inherently thread-safe. Because its internal state is immutable, a single instance can be safely referenced and executed by multiple threads processing different player interactions simultaneously. The responsibility for managing mutable world and entity state is delegated to the `CommandBuffer` and the parent entity processing system, which ensures that world modifications are queued and executed in a safe, deterministic order.

## API Surface
The primary entry point is `tick0`, which is invoked by the server's core interaction module.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(firstRun, time, type, context, cooldownHandler) | void | Variable | Executes the core block placement logic. Validates client data, checks game rules (e.g., range, game mode), and queues world modifications via the CommandBuffer. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `Client`, signifying that this interaction requires synchronized data from the client before execution. |
| configurePacket(packet) | void | O(1) | Populates a network packet with the static configuration of this interaction, such as the block ID to place. |
| needsRemoteSync() | boolean | O(1) | Returns `true`, indicating that this interaction's configuration must be synchronized with the client. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by most game logic developers. It is automatically managed and executed by the server's Interaction System. The system identifies the active interaction for a player (typically based on the held item) and invokes its `tick0` method during the entity update cycle.

A conceptual representation of the system's usage:
```java
// Executed by a high-level InteractionModule
Interaction activeInteraction = player.getActiveInteraction(); // Returns a PlaceBlockInteraction instance

if (activeInteraction instanceof PlaceBlockInteraction) {
    InteractionContext context = buildContextForPlayer(player);
    activeInteraction.tick0(true, deltaTime, type, context, cooldownHandler);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new PlaceBlockInteraction()`. This bypasses the critical data-driven configuration loaded via the CODEC, resulting in an unconfigured and non-functional object.
-   **State Mutation:** Do not attempt to modify the fields of a cached `PlaceBlockInteraction` instance at runtime. These instances are shared across the entire server; modifying one would unpredictably alter the behavior for all subsequent block placements of that type.
-   **Bypassing Context:** Do not call internal helper methods directly. The `tick0` method relies on the complete `InteractionContext` for validation and execution, including entity references, client state, and the command buffer.

## Data Pipeline
The flow of data for a successful block placement is a clear example of the client-server interaction model. The server defines the rules, but the client provides the specific inputs for the action.

> Flow:
> Client Input (Player Clicks) -> Client sends `InteractionSyncData` with position & rotation -> Server Network Layer -> Interaction Module invokes `tick0` on **PlaceBlockInteraction** -> Logic validates range, game mode, and item -> `BlockPlaceUtils.placeBlock` is called -> `CommandBuffer` queues block update in target chunk -> Entity System commits CommandBuffer -> World state is modified and change is replicated to clients.

