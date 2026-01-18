---
description: Architectural reference for CommandManager
---

# CommandManager

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Singleton

## Definition
```java
// Signature
public class CommandManager implements CommandOwner {
```

## Architecture & Concepts
The CommandManager is the central nervous system for all server-side command processing. It functions as a singleton service responsible for the registration, parsing, dispatching, and execution of all text-based commands within the server environment.

Its primary architectural role is to decouple the source of a command (e.g., a player's chat input, a server console entry) from the concrete implementation of the command's logic. It maintains a registry of all available commands, identified by their primary name and a set of optional aliases.

When a command string is received, the CommandManager performs the following sequence:
1.  **Dispatch:** The request is immediately offloaded to a worker thread from the ForkJoinPool to prevent blocking the main server thread.
2.  **Parsing:** The raw string is tokenized into a command name and a list of arguments.
3.  **Resolution:** The command name is used to look up the corresponding AbstractCommand object in its internal registry. Aliases are resolved to their primary command name.
4.  **Execution:** The resolved AbstractCommand is invoked with the parsed arguments and a reference to the CommandSender.
5.  **Feedback:** The result, whether success or failure, is communicated back to the CommandSender, and the entire operation is wrapped in a CompletableFuture to manage its asynchronous lifecycle.

This system provides a unified and extensible entry point for adding new administrative, debugging, or gameplay-related commands to the server.

## Lifecycle & Ownership
- **Creation:** Instantiated once during the server's primary bootstrap sequence. The constructor assigns itself to a static *instance* field, enforcing the singleton pattern.
- **Scope:** The CommandManager is a global, session-scoped service. It persists for the entire lifetime of the server process.
- **Destruction:** The shutdown method is invoked during the server shutdown procedure. This method clears internal collections, primarily the alias map, to assist with garbage collection.

## Internal State & Concurrency
- **State:** The CommandManager maintains mutable state in the form of two primary maps:
    - **commandRegistration:** A map of command names to their corresponding AbstractCommand instances.
    - **aliases:** A map that resolves alias strings to primary command names.
    This state is populated during server startup via the registerSystemCommand method and is intended to be treated as read-only after the initial bootstrap phase.

- **Thread Safety:** This class is **conditionally thread-safe**.
    - **Execution:** Command execution via handleCommand is thread-safe. It internally dispatches work to the ForkJoinPool, ensuring that command logic does not execute on the caller's thread.
    - **Registration:** The registration methods, such as registerSystemCommand, are **not thread-safe**. They perform unsynchronized writes to the internal maps. All command registration must be completed during the single-threaded server initialization phase.

    **WARNING:** Calling registration methods from multiple threads or after the server has started accepting commands will lead to race conditions, likely causing a ConcurrentModificationException or inconsistent behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static CommandManager | O(1) | Returns the global singleton instance. |
| registerSystemCommand(command) | void | O(1) amortized | Registers a command and its aliases. Not thread-safe. |
| handleCommand(sender, command) | CompletableFuture<Void> | O(N) | Asynchronously parses and executes a command string. N is the complexity of the executed command. |
| createVirtualPermissionGroups() | Map | O(C*P) | Aggregates and returns all permission nodes defined by all registered commands. C is command count, P is permissions per command. |
| shutdown() | void | O(A) | Clears internal state during server shutdown. A is the number of aliases. |

## Integration Patterns

### Standard Usage
The standard pattern is to retrieve the singleton instance and submit a command string for execution on behalf of a CommandSender.

```java
// Obtain the global instance
CommandManager manager = CommandManager.get();

// Execute a command for a player
// The future can be used to chain actions after command completion
CompletableFuture<Void> future = manager.handleCommand(player, "/gamemode creative");

future.whenComplete((result, error) -> {
    if (error != null) {
        // Handle command execution failure
    }
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CommandManager()`. This class is a singleton. Direct instantiation will overwrite the static instance, potentially leading to catastrophic state corruption if other systems hold a reference to the previous instance. Always use `CommandManager.get()`.

- **Post-Startup Registration:** Do not call `registerSystemCommand` after the server initialization phase is complete. The registration process is not thread-safe and will cause unpredictable behavior in a live environment.

- **Blocking on the Future:** Do not call `future.get()` or `future.join()` on a critical thread like the main server tick loop. This defeats the purpose of the asynchronous execution model and will cause the server to freeze until the command completes.

## Data Pipeline
The flow of a command from raw text to execution is a well-defined pipeline orchestrated by the CommandManager.

> Flow:
> Raw String Input -> **CommandManager.handleCommand** -> ForkJoinPool Worker -> Tokenizer -> Command & Argument Resolution -> **AbstractCommand Lookup** -> AbstractCommand.acceptCall -> Command Logic Execution -> CompletableFuture Completion -> Feedback to CommandSender

