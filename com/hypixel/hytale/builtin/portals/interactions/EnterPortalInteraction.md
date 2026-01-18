---
description: Architectural reference for EnterPortalInteraction
---

# EnterPortalInteraction

**Package:** com.hypixel.hytale.builtin.portals.interactions
**Type:** Handler / Transient

## Definition
```java
// Signature
public class EnterPortalInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The **EnterPortalInteraction** class is a server-authoritative handler responsible for the logic of a player entering a portal. It is not a standalone system but rather a concrete implementation within the server's Interaction Module. Its role is to be invoked when a player interacts with a block that has been configured to act as a portal entry point.

Architecturally, this class serves as a bridge between a player action, world state, and the instance management system. It encapsulates a multi-stage process:

1.  **Initial Validation:** Performs immediate, synchronous checks to fail fast (e.g., player cooldowns, existence of portal components).
2.  **Asynchronous State Fetching:** Retrieves the state of the destination world without blocking the main world thread. This is critical for performance, as the destination world's data may not be in memory.
3.  **Conditional Logic Execution:** Based on the fetched state, it orchestrates one of several outcomes: teleporting the player, opening a user interface, or sending a failure message.

The class is explicitly server-side, as declared by its `getWaitForDataFrom` method returning **WaitForDataFrom.Server**. This instructs the engine that the client should not predict the outcome of this interaction and must wait for the server's response. This design prevents cheating and ensures a consistent state between the client and server.

## Lifecycle & Ownership
-   **Creation:** An instance of **EnterPortalInteraction** is not created per-interaction. Instead, a single canonical instance is deserialized and instantiated by the server's asset loading system via its static **CODEC** field. This typically occurs at server startup when game configuration files are parsed.
-   **Scope:** The object itself is effectively a stateless singleton managed by the interaction system. It persists for the entire lifetime of the server process. The execution of its methods, however, is scoped to a single player interaction event.
-   **Destruction:** The instance is destroyed when the server shuts down or performs a full reload of its game assets.

## Internal State & Concurrency
-   **State:** This class is fundamentally **stateless and immutable**. It contains no instance fields that store data related to a specific interaction. All required context, such as the world, player entity, and target block, is provided as method arguments.

-   **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, its primary method, **interactWithBlock**, orchestrates operations across different execution contexts and must be handled with extreme care.

    The core of its concurrency model is the use of **CompletableFuture** scheduled on the world's dedicated executor. The initial checks are performed on the main world thread. The potentially blocking call to check the destination world's state is offloaded via `fetchTargetWorldState`. The final logic in the `thenAcceptAsync` callback is crucially scheduled to run back on the original world's thread, ensuring that any mutations to the player or world components are performed safely.

    **WARNING:** Invoking **interactWithBlock** from any thread other than the source world's main thread will lead to race conditions and severe data corruption. The framework guarantees this, but any manual invocation must respect this constraint.

## API Surface
The public contract is defined by its parent, **SimpleBlockInteraction**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Declares the interaction as server-authoritative. Always returns **WaitForDataFrom.Server**. |
| interactWithBlock(...) | void | O(N) / Async | Executes the primary portal entry logic. Complexity is dependent on world data access and is non-blocking. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by game logic developers. It is a system-level handler automatically invoked by the server's **InteractionModule**. A world builder or designer would associate this interaction with a specific block type in a data file.

The system's internal invocation would resemble this conceptual example:

```java
// Engine-level code (conceptual)
// A player interacts with a block at 'targetBlock'
InteractionHandler handler = blockType.getInteractionHandler(); // Returns an EnterPortalInteraction instance

if (handler instanceof EnterPortalInteraction) {
    // The engine populates the context and invokes the handler
    handler.interactWithBlock(world, commandBuffer, type, context, itemInHand, targetBlock, cooldownHandler);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new EnterPortalInteraction()`. The engine relies on the instance created via the **CODEC** for proper registration within the interaction system.
-   **Stateful Subclassing:** Extending this class to add state is an anti-pattern. Interaction handlers are expected to be stateless. Store state in Components or Resources instead.
-   **Blocking Operations:** Modifying this class to perform blocking I/O or long-running computations within the initial phase of **interactWithBlock** will cause server lag. All such operations must be performed asynchronously, following the existing pattern.

## Data Pipeline
The flow of data and control for a successful portal entry is a multi-step, asynchronous process orchestrated by this class.

> Flow:
> Client "Interact" Packet -> Server Network Layer -> InteractionModule -> **EnterPortalInteraction.interactWithBlock** (World Thread) -> **fetchTargetWorldState** (Async Task on World Executor) -> `PortalWorld` Resource Read -> **thenAcceptAsync** Callback (World Thread) -> **InstancesPlugin.teleportPlayerToInstance** -> Entity State Update -> Server Network Layer -> Client "Teleport" Packet

