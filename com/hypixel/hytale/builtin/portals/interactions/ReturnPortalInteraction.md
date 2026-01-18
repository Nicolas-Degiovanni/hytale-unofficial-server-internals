---
description: Architectural reference for ReturnPortalInteraction
---

# ReturnPortalInteraction

**Package:** com.hypixel.hytale.builtin.portals.interactions
**Type:** Transient Handler

## Definition
```java
// Signature
public class ReturnPortalInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts

The ReturnPortalInteraction class is a server-authoritative logic handler responsible for managing a player's exit from a temporary, instanced world (a "Portal World"). It is a concrete implementation of the `SimpleBlockInteraction` contract, designed to be triggered when a player interacts with a specific block type configured to act as a return portal.

Architecturally, this class serves as a stateless rule engine for a single, specific game event. It does not manage the portal block itself, nor does it handle the mechanics of teleportation. Its sole responsibility is to validate the conditions under which a player is *allowed* to exit the instance and, if successful, to issue a command to the appropriate system to perform the exit.

This component is a critical part of a data-driven design pattern. The engine's Interaction Module discovers and invokes this handler based on block configuration assets, not through direct code references. This decouples the core game logic from the specific assets that use it, allowing designers to create return portals without modifying engine code.

Key architectural characteristics include:
*   **Server-Authoritative:** The core logic in `interactWithBlock` runs exclusively on the server. The client-side simulation is explicitly designed to fail and wait for the server's verdict, preventing exploits.
*   **Command-Based:** Rather than directly manipulating player state, it enqueues commands (via the `CommandBuffer`) for other systems to process, such as `InstancesPlugin.exitInstance`. This adheres to a command-query responsibility segregation pattern, ensuring that state changes are managed centrally by the game loop.
*   **Context-Dependent:** It operates on a rich `InteractionContext` provided by the calling system, giving it access to the world, the interacting entity, and other relevant game state without needing to query them globally.

## Lifecycle & Ownership

-   **Creation:** Instances of ReturnPortalInteraction are not created manually using the `new` keyword. The class is designed to be instantiated by the engine's serialization and asset loading systems via its static `CODEC` field. This typically occurs once at server startup when game configurations (e.g., block definitions) are loaded into memory.
-   **Scope:** The configured instance of this class persists for the lifetime of the server, as it is tied to a block's definition. However, the object itself is stateless, and its methods are invoked in a transient, per-interaction scope. No state is carried between invocations.
-   **Destruction:** The configured instance is destroyed during server shutdown when all game assets are unloaded. There is no manual destruction logic.

## Internal State & Concurrency

-   **State:** ReturnPortalInteraction is **stateless and immutable**. It contains no instance fields and all data required for its operations is passed as arguments to its methods. It does not cache data or maintain any state across multiple calls.

-   **Thread Safety:** This class is **conditionally thread-safe**. Because it is stateless, a single configured instance can be safely referenced from multiple threads. However, the `interactWithBlock` method performs mutations on shared game state via the `CommandBuffer`. Therefore, it **must** only be invoked from the main server thread responsible for the corresponding world tick. The engine's Interaction Module guarantees this condition. Calling its methods from an asynchronous task or a different thread will result in severe concurrency issues, such as race conditions and data corruption.

## API Surface

The public API is minimal and conforms to the contract defined by `SimpleBlockInteraction`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) | Server-side entry point. Validates interaction conditions and enqueues an exit command if successful. Mutates the InteractionContext state to indicate failure. |
| simulateInteractWithBlock(...) | void | O(1) | Client-side prediction hook. Always fails immediately, enforcing server authority. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Configuration method that declares this interaction's logic is server-authoritative, forcing the client to wait for a server response. |

## Integration Patterns

### Standard Usage

A developer or designer does not interact with this class directly in Java code. Instead, they associate it with a block in a JSON or similar asset definition file. The engine uses the `CODEC` to link the block to this logic.

*Example Block Configuration (Conceptual)*
```json
{
  "blockName": "hytale:return_portal_core",
  "interaction": {
    "type": "simple_block",
    "handler": "ReturnPortalInteraction"
  }
}
```
When a player interacts with the `hytale:return_portal_core` block, the server's Interaction Module automatically finds and executes the logic within this class.

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ReturnPortalInteraction()`. The class is not designed for manual instantiation and relies on the engine's asset loading pipeline.
-   **Manual Invocation:** Avoid calling `interactWithBlock` directly. This bypasses the engine's Interaction Module, which is responsible for managing critical aspects like cooldowns, permissions, and context setup.
-   **Client-Side Logic:** Do not attempt to replicate this logic on the client. The `getWaitForDataFrom` method explicitly signals that the client must trust the server's state, and the `simulateInteractWithBlock` method is intentionally a no-op.

## Data Pipeline

The flow of data and control for a successful interaction is strictly linear and server-driven.

> Flow:
> Player Input (Client) -> Network Packet (Interaction Event) -> Server Network Layer -> Server Interaction Module -> **ReturnPortalInteraction.interactWithBlock** -> Validation Checks (Player Time in World, World Type) -> CommandBuffer.enqueue(exitInstance) -> Server Game Loop Tick -> Command Processor -> InstancesPlugin executes exit -> Player State Update -> Network Packet (Teleport) -> Client receives update and teleports player.

