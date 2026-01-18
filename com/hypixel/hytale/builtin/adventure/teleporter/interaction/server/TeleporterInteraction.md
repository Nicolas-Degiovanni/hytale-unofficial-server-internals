---
description: Architectural reference for TeleporterInteraction
---

# TeleporterInteraction

**Package:** `com.hypixel.hytale.builtin.adventure.teleporter.interaction.server`
**Type:** Transient Handler

## Definition
```java
// Signature
public class TeleporterInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts

The TeleporterInteraction class is a server-authoritative handler that implements the logic for what occurs when an entity interacts with a teleporter block. It functions as a concrete implementation within a strategy pattern, where the core InteractionModule delegates the handling of a specific block interaction to a specialized class like this one.

This class is fundamentally data-driven. Instances are not created programmatically per-interaction but are deserialized from asset configuration files via the provided static CODEC. This allows game designers to define or modify teleporter behavior, such as particle effects, without changing engine code.

Architecturally, TeleporterInteraction serves as a critical bridge between the player input system and the server's Entity Component System (ECS). Its sole responsibility is to receive an interaction event and translate it into a command that adds a **Teleport** component to the interacting entity. A separate, dedicated `TeleportSystem` will then process this component later in the server tick to execute the actual teleportation. This separation of concerns is a key design principle, ensuring that interaction handlers remain lightweight and focused only on intent, not implementation.

The override of `getWaitForDataFrom` to return `WaitForDataFrom.Server` is a critical architectural choice. It signals to the engine that this interaction is non-predictable and its outcome must be determined exclusively by the server, preventing client-side simulation and potential desynchronization.

## Lifecycle & Ownership

-   **Creation:** A single instance of TeleporterInteraction is instantiated by the server's asset loading system at startup. The `BuilderCodec` is used to deserialize its configuration (e.g., the `particle` field) from a corresponding game asset file, typically a JSON or HOCON file associated with a block definition.
-   **Scope:** Application-scoped. The single, configured instance is registered in a central interaction registry and persists for the entire lifetime of the server process. It is reused for every single teleporter block interaction of its type.
-   **Destruction:** The instance is de-referenced and eligible for garbage collection only when the server is shutting down and the interaction registry is cleared.

## Internal State & Concurrency

-   **State:** This class is effectively stateless regarding individual interactions. Its only field, `particle`, is configured at creation and treated as immutable for the object's lifetime. It does not cache or retain any data from the `interactWithBlock` method calls.

-   **Thread Safety:** This class is **not thread-safe** and must not be invoked from multiple threads concurrently. It is designed to be called exclusively from a world's main update thread. The use of a `CommandBuffer` for all world mutations is the mechanism that ensures engine-level safety. All component additions are queued in the buffer and applied at a designated synchronization point in the server tick, preventing race conditions within the ECS.

## API Surface

The public API is minimal, consisting primarily of overrides to fulfill the `SimpleBlockInteraction` contract. The core logic resides in `interactWithBlock`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) Amortized | Processes the interaction event. Validates the state of the teleporter and the interacting entity, then adds a `Teleport` component to the entity via the `CommandBuffer` to schedule a teleport. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `Server`, enforcing that this interaction is server-authoritative and cannot be predicted by the client. |

## Integration Patterns

### Standard Usage

This class is not intended to be instantiated or called directly by game logic developers. It is configured as part of a block's properties and invoked automatically by the server's `InteractionModule`.

A typical data-driven configuration in a block asset file might look like this:

```json
// Example: my_teleporter.json (Conceptual)
{
  "blockClass": "...",
  "interaction": {
    "type": "TeleporterInteraction",
    "particle": "hytale:effects_particles_teleport_poof"
  }
}
```

The engine's interaction system would then invoke the handler polymorphically.

```java
// HYPOTHETICAL: Engine code within an InteractionModule
// This code is conceptual and illustrates how the engine uses the class.

void onPlayerInteract(Player player, Vector3i blockPosition) {
    World world = player.getWorld();
    BlockType blockType = world.getBlockTypeAt(blockPosition);
    
    // The engine retrieves the pre-configured handler for this block type
    SimpleBlockInteraction handler = blockType.getInteractionHandler();

    if (handler != null) {
        // The engine dispatches the event to the correct handler
        handler.interactWithBlock(world, ...);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new TeleporterInteraction()`. Doing so bypasses the critical configuration-driven setup via the `CODEC`, resulting in a handler with null properties (like `particle`) and one that is not registered with the engine.
-   **Stateful Implementation:** Do not add mutable fields to this class to track interaction state. A single instance is shared across all interactions of this type, and adding state would introduce severe, unpredictable bugs and race conditions.
-   **Client-Side Invocation:** This class is part of the server JAR and relies on server-side ECS components. Attempting to use it on the client will result in a crash.

## Data Pipeline

The flow of data for a successful teleporter interaction is strictly linear and server-authoritative. The class acts as an initial processing and validation step that triggers a larger, asynchronous ECS-based workflow.

> Flow:
> Client Input (Interaction) -> Network Packet -> Server `InteractionModule` -> **TeleporterInteraction.interactWithBlock** -> `CommandBuffer` queues `addComponent(Teleport)` -> `TeleportSystem` (later in tick) processes `Teleport` component -> Entity `TransformComponent` is updated -> Network sync to clients -> Client renders entity at new position

