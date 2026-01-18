---
description: Architectural reference for WhitelistAddCommand
---

# WhitelistAddCommand

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.commands
**Type:** Transient

## Definition
```java
// Signature
public class WhitelistAddCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The WhitelistAddCommand class is a concrete implementation of the Command Pattern, designed to operate within the server's asynchronous command execution framework. It extends AbstractAsyncCommand, signaling that its primary operation involves I/O-bound work that must not block the main server thread.

Its role is to serve as the logical endpoint for the `/whitelist add` console command. It encapsulates the entire workflow for this action: parsing arguments, performing an external identity lookup, modifying a persistent state provider, and reporting results back to the command issuer.

Architecturally, this class decouples the command input system from the access control module. It depends on two key services injected via its constructor and method parameters:
1.  **HytaleWhitelistProvider:** The authoritative source for whitelist data. This command acts as a client to this service.
2.  **AuthUtil:** A utility for resolving player-friendly usernames into system-critical UUIDs. This interaction represents a significant external dependency, likely involving a network request to a central authentication service, which is the primary reason for the command's asynchronous nature.

## Lifecycle & Ownership
-   **Creation:** An instance of WhitelistAddCommand is created during the server's bootstrap phase, typically within the initialization block of a parent module responsible for access control. The required HytaleWhitelistProvider dependency is injected at this time. It is then registered with a central CommandManager service.
-   **Scope:** The object instance persists for the entire server session. It is a long-lived object that handles all invocations of the `/whitelist add` command.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The internal state of the WhitelistAddCommand instance is immutable after construction. It holds final references to the HytaleWhitelistProvider and its argument definition (usernameArg). The class itself is stateless with respect to individual command executions; all necessary context is provided via the CommandContext parameter in the executeAsync method.
-   **Thread Safety:** This class is thread-safe. Its immutable state prevents race conditions within the command object itself. The executeAsync method is designed to be invoked by the command system's dedicated thread pool. The use of CompletableFuture ensures that the I/O-bound UUID lookup and subsequent whitelist modification occur off the main server thread, preventing stalls. Any state modification is delegated to the HytaleWhitelistProvider, which is assumed to be internally thread-safe.

## API Surface
The primary contract is the overridden `executeAsync` method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | I/O Bound | Executes the command logic. Resolves the username argument to a UUID via a network call, then attempts to add it to the whitelist. Reports success, failure, or "already exists" status to the user via the CommandContext. |

## Integration Patterns

### Standard Usage
A developer does not invoke this class directly. It is instantiated and registered with the server's command system, which then dispatches incoming commands to it.

```java
// Example of how the system registers this command during startup
HytaleWhitelistProvider provider = context.getService(HytaleWhitelistProvider.class);
CommandManager commandManager = context.getService(CommandManager.class);

// The command is constructed with its dependency and registered
commandManager.register(new WhitelistAddCommand(provider));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Never call `new WhitelistAddCommand(...).executeAsync(ctx)` manually. The command system is responsible for managing the command lifecycle, thread execution, and context propagation. Bypassing the registry can lead to unpredictable behavior.
-   **Blocking on the Future:** Do not call `.join()` or `.get()` on the CompletableFuture returned by `executeAsync` from a synchronous context, such as the main server thread. This will block the thread, freeze the server, and completely negate the benefits of the asynchronous design.

## Data Pipeline
This command processes data in a clear, sequential, and asynchronous flow.

> Flow:
> User Input (`/whitelist add <username>`) -> Command Parser -> **WhitelistAddCommand.executeAsync** -> AuthUtil (External Network API Call) -> UUID Result -> HytaleWhitelistProvider.modify -> Filesystem/Database Write -> CommandContext.sendMessage -> User Feedback
---

