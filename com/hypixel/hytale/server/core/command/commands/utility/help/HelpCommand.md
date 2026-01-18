---
description: Architectural reference for HelpCommand
---

# HelpCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.help
**Type:** Transient Command Object

## Definition
```java
// Signature
public class HelpCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The HelpCommand class is a server-side command implementation that provides players with information about available commands. It serves as a critical bridge between the server's Command System and the player's UI System.

Architecturally, it is designed as an asynchronous command by extending **AbstractAsyncCommand**. This is a deliberate design choice to prevent server stalls. When a player executes the help command, the work of resolving command names and preparing the UI is performed without blocking the main server thread. The result, a request to open a UI page, is then safely scheduled for execution on the appropriate world thread.

The command supports two primary forms: a general help request which displays a list of all available commands, and a specific request which provides details for a single command. This is implemented using a nested **HelpCommandVariant** class, a common pattern in the command system for handling sub-commands or different argument structures without cluttering the parent command.

Interaction with the core server components is central to its operation:
- It queries the **CommandManager** to resolve command names and aliases.
- It uses the **CommandContext** to identify the sender and retrieve their entity reference (**PlayerRef**).
- It interacts with the player's **PageManager** component to open the **CommandListPage**, a specialized UI for displaying command information.

## Lifecycle & Ownership
- **Creation:** A single instance of HelpCommand is created by the **CommandManager** during the server's bootstrap phase. The command is discovered and registered automatically, making it available for the entire server session.
- **Scope:** The HelpCommand object is a long-lived service object. It persists from the moment it is registered until the server shuts down. It is not instantiated per-execution.
- **Destruction:** The object is dereferenced and eligible for garbage collection only when the **CommandManager** is cleared during a server shutdown.

## Internal State & Concurrency
- **State:** The HelpCommand object is effectively stateless and immutable after its initial construction. It contains no mutable instance fields. All state required for an execution, such as the player sending the command or the arguments provided, is encapsulated within the **CommandContext** object passed to the execution method.

- **Thread Safety:** This class is thread-safe. As an **AbstractAsyncCommand**, its primary execution logic runs asynchronously. The method **executeAsync** returns a **CompletableFuture**, signaling that the work will be completed later. Crucially, any operations that must modify game state, such as opening a UI for a player, are explicitly scheduled to run on that player's world thread via `CompletableFuture.runAsync(..., world)`. This pattern of off-thread computation followed by on-thread mutation is fundamental to the server's concurrency model and prevents race conditions.

## API Surface
The primary API is not programmatic but rather the command interface exposed to players in the game.

| Command Syntax | Arguments | Complexity | Description |
| :--- | :--- | :--- | :--- |
| /help | (none) | O(N) | Opens a UI listing all commands available to the player. Complexity is relative to the number of registered commands. |
| /help `[command]` | `command`: String | O(N) | Resolves the provided command name or alias and opens the help UI focused on that specific command. |
| /? | (none) | O(N) | An alias for the base /help command. |

## Integration Patterns

### Standard Usage
A player executes the command through the in-game chat interface. The system parses this input and dispatches it to the registered **HelpCommand** instance.

```
// Player Input in chat
/help kick

// Conceptual server-side dispatch
CommandContext context = createFromPlayerInput("/help kick");
CommandManager manager = server.getCommandManager();
AbstractCommand command = manager.findCommand("help");

// The system invokes the asynchronous execution
command.executeAsync(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new HelpCommand()`. The command system relies on a single, registered instance managed by the **CommandManager**. Manually creating an instance will result in a non-functional command object that is disconnected from the server.
- **Synchronous Assumption:** Do not call **executeAsync** and expect the UI to be open immediately in the next line of code. The method returns a **CompletableFuture** and the operation completes on a different thread at a later time.
- **Console Execution:** Attempting to execute this command from the server console will have no effect. The implementation explicitly checks if the sender is a player (`context.isPlayer()`) before attempting to open a UI, as a UI cannot be rendered for a non-player entity.

## Data Pipeline
The flow of data for a specific help request follows a clear, asynchronous path from player input to UI update.

> Flow:
> Player Chat Input (`/help kick`) -> Network Layer -> Command Parser -> **CommandManager** dispatches to **HelpCommand** -> **executeAsync** returns a **CompletableFuture** -> Async task resolves command name -> Task is scheduled on the World thread -> **PageManager.openCustomPage** is called -> UI System sends update packet to Client -> Client renders the **CommandListPage** UI.

