---
description: Architectural reference for TeleportWorldCommand
---

# TeleportWorldCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport
**Type:** Transient

## Definition
```java
// Signature
public class TeleportWorldCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The **TeleportWorldCommand** class is a server-side command handler responsible for processing player-initiated requests to teleport to a different world's spawn point. It serves as a critical bridge between the high-level Command System and the low-level Entity Component System (ECS).

Architecturally, this command follows a decoupled, intent-based design. Instead of directly manipulating the player's **TransformComponent**, its primary function is to validate the request and then attach a **Teleport** component to the player entity. A separate, dedicated system (e.g., a **TeleportSystem**) is responsible for detecting this component later in the server tick and executing the actual teleportation logic. This pattern ensures that complex state changes like teleportation are handled atomically by a specialized system, preventing race conditions and keeping command handlers lightweight and focused on input processing.

This class interacts with several core engine modules:
*   **Universe:** To resolve world names and retrieve **World** objects.
*   **Command System:** To define command arguments, permissions, and receive the execution context.
*   **Entity Component System (Store):** To read the player's current state (**TransformComponent**, **HeadRotation**) and write its intent to teleport (**Teleport** component) and update their history (**TeleportHistory**).

## Lifecycle & Ownership
-   **Creation:** An instance of **TeleportWorldCommand** is created by the server's command registration service during the bootstrap phase. It is not intended for manual instantiation by developers.
-   **Scope:** The object persists for the entire server session, held by the central command registry. It does not hold per-player state, making a single instance reusable for all command invocations.
-   **Destruction:** The instance is garbage collected when the command registry is cleared, typically during server shutdown.

## Internal State & Concurrency
-   **State:** This class is effectively stateless regarding its execution logic. It contains a final field, **worldNameArg**, which defines the command's argument structure. This configuration is immutable after the constructor is called. All state required for execution (**CommandContext**, **Store**, **Ref**) is passed as method parameters to the **execute** method.

-   **Thread Safety:** **This class is not thread-safe and must only be invoked from the main server thread.** The **execute** method directly interacts with the ECS **Store**, which is not designed for concurrent access from multiple threads. The engine's command dispatcher guarantees that all command executions are serialized and processed on the appropriate game loop thread.

    **Warning:** Manually invoking the **execute** method from an asynchronous task or a different thread will lead to data corruption, concurrency exceptions, and unpredictable server behavior.

## API Surface
The public API is minimal, designed exclusively for consumption by the server's command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(log N) | Processes the teleport request. Complexity is dominated by the world lookup (N = number of worlds). Throws exceptions on invalid context. |

## Integration Patterns

### Standard Usage
This class is not designed for direct programmatic invocation. It is automatically discovered and executed by the server's command processing pipeline when a player types the corresponding command into chat.

The flow is managed entirely by the engine:
1.  Player enters: `/teleport world <worldName>`
2.  The server's command parser identifies **TeleportWorldCommand** as the handler.
3.  The parser validates and extracts the *worldName* argument.
4.  The engine invokes the **execute** method on the registered instance, passing the full execution context.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new TeleportWorldCommand()`. The command system handles instantiation and registration. Manual creation results in an object that is not connected to the command parser or permission system.
-   **Manual Execution:** Avoid calling the **execute** method directly. Doing so bypasses critical framework features like permission checks, argument parsing, and context setup, potentially leaving the system in an inconsistent state.
-   **State Modification:** Do not attempt to modify the internal **worldNameArg** field via reflection after construction. This can break the command's contract with the command parser.

## Data Pipeline
The data flow for a successful world teleport command demonstrates the separation of concerns between command handling and system processing.

> Flow:
> Player Chat Input -> Network Packet -> Server Command Parser -> **TeleportWorldCommand.execute()** -> ECS State Change (adds **Teleport** component) -> TeleportSystem (processes component) -> Player **TransformComponent** Update -> Network Sync -> Client Position Update

