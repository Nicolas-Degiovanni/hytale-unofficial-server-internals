---
description: Architectural reference for BackupCommand
---

# BackupCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility
**Type:** Transient

## Definition
```java
// Signature
public class BackupCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The **BackupCommand** class is a concrete implementation within the server's Command System. It serves as the user-facing entry point for initiating a full backup of the server's **Universe**. This class is not responsible for the backup logic itself; instead, it acts as a lightweight adapter that bridges user input to the core server services.

Its primary responsibilities are:
1.  **Input Registration:** Registers the command name "backup" with the server's command dispatcher.
2.  **Pre-condition Validation:** Performs critical safety checks to ensure the server is in a valid state for a backup operation. This includes verifying that the server has fully booted and that a backup directory has been configured in the server options.
3.  **Delegation:** If validation passes, it delegates the complex and potentially long-running backup operation to the **Universe** singleton.
4.  **User Feedback:** Provides immediate feedback to the command issuer, notifying them that the backup is starting and, upon asynchronous completion, that it has finished.

This class adheres to the Command pattern, encapsulating a request as an object, thereby decoupling the requester of an action from the object that performs the action.

### Lifecycle & Ownership
-   **Creation:** A single instance of **BackupCommand** is instantiated by the Command System during server bootstrap. The system scans for command implementations and registers them in a central registry.
-   **Scope:** The object instance is a singleton within the context of the command registry and persists for the entire server session.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection during server shutdown when the Command System is dismantled.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. The **Message** fields are static, final, and immutable, defined at compile time. No mutable instance state is maintained between invocations of its execution method.
-   **Thread Safety:** The class is inherently thread-safe due to its stateless nature. The primary execution method, **executeAsync**, is designed for an asynchronous environment. It returns a **CompletableFuture**, offloading the blocking backup work from the main command processing thread. This prevents the server from stalling during a potentially lengthy backup operation. Callbacks for sending completion messages are safely chained to the future, ensuring they execute only after the backup process concludes.

## API Surface
The public contract is defined by its inheritance from **AbstractAsyncCommand**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BackupCommand() | constructor | O(1) | Registers the command name and description with the superclass. |
| executeAsync(CommandContext) | CompletableFuture<Void> | O(1) | Initiates the backup process. The method itself returns immediately, but it triggers a long-running background task. |

## Integration Patterns

### Standard Usage
This command is intended to be invoked by a server administrator through the server console or an in-game chat interface. The Command System handles parsing, routing, and invocation.

```java
// This code is conceptual and represents how the command system
// would invoke the command. Direct invocation is an anti-pattern.

// 1. User types "/backup"
// 2. Command system parses and finds the registered BackupCommand instance.
CommandContext context = create_context_for_user();
BackupCommand cmd = commandRegistry.get("backup");

// 3. System invokes the command.
cmd.executeAsync(context);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BackupCommand()` in application code. The command will not be registered with the server and will have no effect. The only valid creator is the Command System during its initialization phase.
-   **Synchronous Blocking:** Do not attempt to block on the returned **CompletableFuture** from a critical server thread, such as the main game loop. Doing so would defeat the purpose of the asynchronous design and cause the server to freeze until the backup is complete.

## Data Pipeline
**BackupCommand** primarily manages a control flow rather than a data transformation pipeline. It translates a user action into a system-level operation.

> Flow:
> User Input (`/backup`) -> Command Parser -> Command Dispatcher -> **BackupCommand.executeAsync** -> Validation Checks -> **Universe.runBackup()** -> Filesystem I/O -> **CompletableFuture** Completion -> CommandContext.sendMessage("...complete") -> User Console

The command itself is the trigger, while the core data handling (reading world files, writing to an archive) is managed entirely by the **Universe** service.

