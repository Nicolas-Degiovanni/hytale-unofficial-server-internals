---
description: Architectural reference for UnbanCommand
---

# UnbanCommand

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.commands
**Type:** Transient Command Handler

## Definition
```java
// Signature
public class UnbanCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The UnbanCommand class is a server-side command handler responsible for removing a player's ban from the server. It serves as a direct interface between a privileged user (e.g., a server administrator) and the server's access control module.

Architecturally, this class embodies several key engine patterns:
-   **Command Pattern:** It encapsulates a specific server action (unbanning a player) into a standalone object. This object is discovered and executed by a central Command System.
-   **Asynchronous Execution:** By extending AbstractAsyncCommand, it signals to the command system that its work is non-blocking and should not be performed on the main server thread. This is critical because its operation involves a network call to resolve a player's UUID, which is an I/O-bound task with unpredictable latency.
-   **Dependency Injection:** The command receives its primary dependency, the HytaleBanProvider, via its constructor. This decouples the command logic from the underlying data storage mechanism for bans, which could be a file, a database, or a remote service.

The command's primary responsibility is to orchestrate the unban process: resolve the target player's identity, verify they are currently banned, instruct the data provider to remove the ban, and report the outcome to the command issuer.

### Lifecycle & Ownership
-   **Creation:** Instantiated once at server startup by a higher-level authority, typically a CommandRegistry or a module loader responsible for access control features. The HytaleBanProvider dependency is injected during this phase.
-   **Scope:** The UnbanCommand object is a long-lived singleton for the duration of the server's runtime. It is registered with the command system and remains available to be executed at any time.
-   **Destruction:** The object is dereferenced and eligible for garbage collection during server shutdown when the CommandRegistry is cleared.

## Internal State & Concurrency
-   **State:** The UnbanCommand is effectively stateless with respect to its execution. It holds immutable references to the HytaleBanProvider and its argument definition (usernameArg), which are configured at construction. All state required for an operation is provided transiently through the CommandContext object passed to the executeAsync method.
-   **Thread Safety:** This class is thread-safe. Its internal fields are final. The executeAsync method returns a CompletableFuture, ensuring that the potentially long-running logic is executed on a separate thread pool managed by the server's asynchronous task scheduler. All state modification is delegated to the HytaleBanProvider, which is assumed to be internally synchronized. Multiple unban operations can be initiated concurrently without risk of race conditions within this class.

## API Surface
The public contract is primarily for the command system and dependency injection framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UnbanCommand(HytaleBanProvider) | constructor | O(1) | Constructs the command, injecting the data provider for ban management. |
| executeAsync(CommandContext) | CompletableFuture<Void> | O(Network + Storage) | The command's entry point. Asynchronously resolves the username, modifies the ban list, and sends feedback. |

## Integration Patterns

### Standard Usage
This command is not intended for direct invocation in code. It is automatically executed by the server's command handler when a user with appropriate permissions types the command into the server console or chat.

The conceptual flow within the command system is as follows:
```java
// PSEUDOCODE: How the command system invokes this class
CommandContext context = commandSystem.createContextFor("/unban PlayerName");
UnbanCommand command = commandSystem.getCommand("unban");

// The system invokes the method, handling the future's completion
command.executeAsync(context);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new UnbanCommand()`. The command will not be registered with the server and will lack its required HytaleBanProvider dependency.
-   **Synchronous Blocking:** Do not call `.join()` or `.get()` on the returned CompletableFuture from the main server thread. This defeats the purpose of an asynchronous command and will cause the server to hang during the UUID lookup.
-   **Manual Invocation:** Avoid calling `executeAsync` directly. The CommandContext is a complex object that should be constructed and managed by the server's command processing pipeline to ensure the source, permissions, and reply channels are correctly configured.

## Data Pipeline
The flow of data for a typical unban operation is linear, moving from user input to data store modification.

> Flow:
> User Input (`/unban PlayerName`) -> Server Command Parser -> Command Dispatcher -> **UnbanCommand**.executeAsync -> AuthUtil (Network UUID Lookup) -> HytaleBanProvider.modify (Data Store Write) -> CommandContext.sendMessage (Feedback to User)

