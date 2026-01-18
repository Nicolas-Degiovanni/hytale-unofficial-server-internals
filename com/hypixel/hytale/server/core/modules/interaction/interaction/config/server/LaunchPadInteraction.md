---
description: Architectural reference for LaunchPadInteraction
---

# LaunchPadInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Configuration Object

## Definition
```java
// Signature
public class LaunchPadInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The **LaunchPadInteraction** class is a server-authoritative, data-driven behavior definition that implements the logic for launch pad blocks. It is not a long-lived service but rather a stateless configuration object deserialized by the engine's codec system.

Architecturally, it serves as a specific implementation within the broader server Interaction Module. When an entity's interaction with a block is processed, the engine resolves the block's configured interaction type to an instance of this class. Its primary responsibility is to translate the interaction event into a set of commands to be executed against the world state.

This class operates entirely within the server's Entity Component System (ECS) framework. It does not modify world state directly. Instead, it submits instructions—such as changing an entity's velocity or spawning particles—to a **CommandBuffer**. This pattern ensures that all state mutations are deferred and processed in a deterministic order at a later stage of the game tick, which is critical for system stability and concurrency management.

A key design choice is its declaration of `WaitForDataFrom.Server`. This explicitly instructs the client-side prediction system to *not* simulate this interaction. The client must wait for the server's authoritative state update, preventing desynchronization where a player might incorrectly predict being launched.

## Lifecycle & Ownership
-   **Creation:** Instances are not created manually via a constructor. They are materialized by the **BuilderCodec** system when the server loads game asset configurations from disk. Each unique launch pad definition in the game's configuration files will correspond to a deserialized instance of this class.
-   **Scope:** An instance is effectively a stateless singleton for its corresponding configuration. It persists as long as the game assets are loaded, but it holds no per-interaction state. Its methods can be considered pure functions of their inputs.
-   **Destruction:** Managed by the Java garbage collector when game assets are unloaded. No explicit cleanup is required.

## Internal State & Concurrency
-   **State:** This class is **immutable and stateless**. It contains no member fields for storing data, and all necessary context is provided as arguments to its methods. This design makes it highly predictable and reusable.
-   **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless nature, a single instance can be safely invoked by multiple worker threads simultaneously without risk of race conditions or data corruption. Thread safety of the overall interaction process is managed at a higher level by the engine, primarily through the **CommandBuffer** mechanism.

## API Surface
The primary contract is with the interaction dispatch system via its overridden protected methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns a constant indicating that this interaction is server-authoritative and should not be predicted by the client. |
| interactWithBlock(...) | void | O(log N) | Core server-side logic. Applies velocity changes and particle effects. Complexity is dominated by the spatial query for nearby players. |
| simulateInteractWithBlock(...) | void | O(1) | No-op. Intentionally left empty to enforce the server-authoritative model defined by getWaitForDataFrom. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by gameplay programmers. It is invoked exclusively by the server's **InteractionModule** as part of the game tick's entity processing phase. The system identifies an interaction, resolves it to this handler, and dispatches the call.

```java
// Conceptual example of engine-level dispatch
InteractionType interactionType = ...;
InteractionContext context = ...;
Vector3i targetBlock = ...;

// The engine resolves the block's behavior to a LaunchPadInteraction instance
SimpleBlockInteraction handler = getInteractionHandlerForBlock(targetBlock);

if (handler instanceof LaunchPadInteraction) {
    // The engine invokes the handler with the current world state and command buffer
    handler.interactWithBlock(world, commandBuffer, interactionType, context, ...);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new LaunchPadInteraction()`. The behavior of this class is defined by data that is only supplied during codec-based deserialization. Manual instantiation creates a non-functional object.
-   **External Invocation:** Do not call `interactWithBlock` from outside the engine's core interaction loop. The method relies on a specific state (e.g., a valid and open **CommandBuffer**) that is only guaranteed during the correct phase of the server tick.
-   **Client-Side Usage:** This is a server-only class. Attempting to use it on the client will fail and violates the netcode design.

## Data Pipeline
The flow of data and control for a launch pad interaction is strictly ordered and server-driven.

> Flow:
> Entity Collision/Interaction Event -> Server InteractionModule -> **LaunchPadInteraction**.interactWithBlock -> Write `ChangeVelocity` instruction to **CommandBuffer** -> Tick Processor executes **CommandBuffer** -> Entity **Velocity** component is updated -> State change is replicated to clients

