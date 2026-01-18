---
description: Architectural reference for SudoCommand
---

# SudoCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Command Handler

## Definition
```java
// Signature
public class SudoCommand extends CommandBase {
```

## Architecture & Concepts
The SudoCommand class is a server-side command handler that implements the Command Pattern within the Hytale server architecture. Its primary function is to enable a privileged user (the command sender) to execute a command *as if* it were sent by another player, or by all players on the server.

This class acts as a high-level proxy and dispatcher. It does not contain any game logic itself. Instead, it parses its arguments to identify a target player (or players) and a subordinate command string. It then leverages core server systems—specifically the **Universe** for player lookups and the **World** scheduler for thread-safe execution—to re-inject the subordinate command into the **CommandManager** from the perspective of the target player.

This design decouples the act of command impersonation from command execution, ensuring that the final command is processed through the same pipeline as a normal player-sent command, including any associated permission checks or context-specific logic relevant to the target player.

### Lifecycle & Ownership
- **Creation:** A single instance of SudoCommand is created by the **CommandManager** during the server's bootstrap phase. The manager scans for all classes extending **CommandBase** and instantiates them for registration.
- **Scope:** The instance is a stateless singleton for the lifetime of the server. It persists from server start to shutdown.
- **Destruction:** The object is de-referenced and eligible for garbage collection only when the server is shutting down and the **CommandManager** clears its command registry.

## Internal State & Concurrency
- **State:** SudoCommand is designed to be stateless. All necessary information for an execution is provided via the **CommandContext** argument in the executeSync method. The only internal fields are final definitions for command arguments, which are immutable after construction.

- **Thread Safety:** This class is thread-safe due to its stateless nature. However, its interaction with the game world requires careful thread management. The core of its concurrency model is the call to `world.execute`. This is a critical safety mechanism. It ensures that the final command dispatch, which may modify entity or world state, is scheduled to run on the dedicated thread for that player's **World**. This prevents race conditions and guarantees that all game state modifications occur within the server's main tick loop for that world.

    **WARNING:** Any direct modification of game state within executeSync would be a severe violation of the server's threading model. All world-mutating logic is correctly deferred to the world's task scheduler.

## API Surface
The public API is inherited from **CommandBase** and invoked exclusively by the **CommandManager**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(N) | Parses sender input, finds N target players, and schedules a new command for each on their respective world thread. Sends feedback messages to the original sender's context. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly via code. It is a command handler invoked by the server when a player or the console types the command.

```
// Player or Console Input
/sudo <player_name | *> <command_to_run>

// Example: Make the player "Notch" say "Hello World"
/sudo Notch say Hello World

// Example: Make all players run the /who command
/sudo * who
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SudoCommand()`. The command system relies on a single, registered instance managed by the **CommandManager**. Creating a new instance will have no effect and will not be registered to handle commands.
- **Direct Invocation:** Do not call the `executeSync` method directly. This bypasses the **CommandManager**'s parsing, permission system, and initial context setup. All commands must flow through the central **CommandManager.handleCommand** method.
- **Assuming Synchronous Execution:** The "Sync" in `executeSync` refers to the context of the *caller's* thread. The actual command executed by the target player is scheduled *asynchronously* on a different thread (the world thread). Code cannot assume the sudo'd command has completed after `executeSync` returns.

## Data Pipeline
The data flow for this command involves parsing, redirection, and re-injection into the server's command processing system.

> Flow:
> Admin Raw Input (`/sudo Steve /help`) -> **CommandManager** (Routing) -> **SudoCommand**.executeSync -> **Universe**.getPlayer("Steve") -> Target Player's **World**.execute(lambda) -> (On World Thread) **CommandManager**.handleCommand(Steve, "/help") -> **HelpCommand**.executeSync

