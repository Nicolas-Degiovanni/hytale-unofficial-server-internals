---
description: Architectural reference for HideCommand
---

# HideCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Command Definition

## Definition
```java
// Signature
public class HideCommand extends AbstractCommandCollection {
```

## Architecture & Concepts

The HideCommand class is a container that implements the **Composite Pattern** for server-side command management. It does not contain any execution logic itself; instead, it serves as a root entry point that aggregates and organizes a suite of related sub-commands for controlling player visibility. Its primary role is to define the `hide` command namespace and route execution to the appropriate specialized command handler.

This class acts as a bridge between the server's command parsing system and the underlying player visibility logic, which is managed by the HiddenPlayersManager component on each player entity. By centralizing the command definitions, it provides a single, coherent interface for administrators and players to manage who can see whom within the game worlds.

A key architectural aspect is the mixed-threading model employed by its sub-commands:

*   **Asynchronous Commands:** Operations targeting specific players (HidePlayerCommand, ShowPlayerCommand) extend AbstractAsyncCommand. This offloads potentially blocking operations, such as player lookups, from the main server thread, ensuring server responsiveness. The final state mutation is then marshaled back to the appropriate world thread.
*   **Synchronous Dispatch Commands:** Global operations (HideAllCommand, ShowAllCommand) extend CommandBase for synchronous execution. However, they do not perform heavy work on the calling thread. Instead, they iterate through all active worlds and dispatch a task to each world's dedicated execution queue via `world.execute`. This "fan-out" pattern is critical for concurrency safety, as it guarantees that all modifications to a world's entities occur on its designated thread, preventing race conditions and data corruption.

## Lifecycle & Ownership

-   **Creation:** HideCommand is instantiated once at server startup by the central CommandRegistry. The registry scans the classpath for command definitions and registers them for use.
-   **Scope:** Application-scoped. A single instance of HideCommand persists for the entire lifetime of the server process.
-   **Destruction:** The object is de-referenced and eligible for garbage collection only when the server is shutting down and the CommandRegistry is cleared.

## Internal State & Concurrency

-   **State:** The HideCommand class is effectively **immutable** after construction. Its internal state consists of a list of its sub-commands, which is populated in the constructor and never modified thereafter. The inner command classes are themselves stateless, operating exclusively on the CommandContext provided for each execution.
-   **Thread Safety:** The class is inherently thread-safe. The command system guarantees that the execution logic within its sub-commands is invoked according to a strict threading model. All interactions with the game state, such as modifying a player's hidden player list, are safely dispatched to the relevant world thread. Direct, multi-threaded access to a single command instance is not a valid use case and is prevented by the command system's design.

## API Surface

The primary API is the command-line interface exposed to users, not a programmatic one for developers. The class structure defines the following command variants:

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hide `player` `[target]` | Async | O(N) | Hides a player from a specific target or from all other players. N is the number of players if target is omitted. |
| hide all | Sync Dispatch | O(W * P^2) | Hides every player from every other player across all worlds. W is worlds, P is players per world. |
| show `player` `[target]` | Async | O(N) | Makes a player visible to a specific target or to all other players. |
| showall | Sync Dispatch | O(W * P^2) | Makes every player visible to every other player across all worlds. |

**WARNING:** The complexity of `hide all` and `showall` is high but distributed. The work is fanned out across all world threads, so the operation does not block the main server thread.

## Integration Patterns

### Standard Usage

This class is not intended for direct programmatic use. It is automatically discovered and managed by the server's command system. A user, such as a server administrator or a player with appropriate permissions, interacts with it via the in-game chat or server console.

```sh
# Hides PlayerA from PlayerB's view
/hide PlayerA PlayerB

# Hides PlayerA from everyone
/hide PlayerA

# Makes all players invisible to each other
/hide all
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new HideCommand()`. The command system handles instantiation and registration. Creating an instance manually will have no effect on the server.
-   **Direct Method Invocation:** Never call the `executeSync` or `executeAsync` methods on the inner command classes directly. Doing so bypasses the entire command processing pipeline, including argument parsing, permission checks, and the critical thread-safety mechanisms. All command execution must flow through the server's central command dispatcher.

## Data Pipeline

The flow of data for a typical command execution is a multi-stage process that ensures safety and correctness, moving from the network layer to the core game state.

> Flow:
> User Input (`/hide PlayerA`) -> Network Layer -> Server Command Parser -> **HideCommand** (Router) -> HidePlayerCommand (Executor) -> Argument Resolvers (Converts "PlayerA" to PlayerRef) -> Async Task Scheduler -> World Thread Executor -> PlayerRef.getHiddenPlayersManager().hidePlayer(uuid) -> Entity State Change -> Visibility Packet Sent to Clients

