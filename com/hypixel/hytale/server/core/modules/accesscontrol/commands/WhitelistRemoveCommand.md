---
description: Architectural reference for WhitelistRemoveCommand
---

# WhitelistRemoveCommand

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.commands
**Type:** Component / Command Object

## Definition
```java
// Signature
public class WhitelistRemoveCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The WhitelistRemoveCommand encapsulates the server-side logic for removing a player from the server's whitelist. It is an implementation of the **Command Pattern**, designed to be registered with and invoked by the server's central command processing system.

Architecturally, this class serves as a non-blocking bridge between three distinct systems:
1.  The **Command System**, which provides the invocation context and user-facing interface.
2.  The **Authentication Service**, an external dependency used to resolve player usernames into unique identifiers (UUIDs).
3.  The **Whitelist Provider**, which manages the persistent state of the whitelist.

By extending AbstractAsyncCommand, this class signals to the engine that its execution may involve long-running or I/O-bound operations. The use of CompletableFuture ensures that the primary server thread is not blocked during the external network call to resolve the player's UUID, which is critical for maintaining server performance and responsiveness.

The class decouples itself from the underlying whitelist storage mechanism (e.g., a JSON file, a database) by operating exclusively through the HytaleWhitelistProvider interface. This adheres to the Dependency Inversion Principle, making the command logic independent of the data storage implementation.

### Lifecycle & Ownership
-   **Creation:** A single instance of WhitelistRemoveCommand is instantiated during server startup, typically by its parent module (e.g., AccessControlModule). The required HytaleWhitelistProvider dependency is injected via the constructor at this time.
-   **Scope:** The object is long-lived. It is registered with the command dispatcher and persists for the entire server session.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively stateless regarding its execution logic. It holds immutable references to its dependencies (the whitelist provider) and its argument definitions. All mutable state related to the whitelist itself is managed externally by the HytaleWhitelistProvider.

-   **Thread Safety:** This class is thread-safe. Each invocation of executeAsync operates on a unique CommandContext. The core logic, including the network call and subsequent data modification, is executed on a separate thread pool managed by the CompletableFuture framework. Any potential race conditions related to modifying the whitelist data are the responsibility of the injected HytaleWhitelistProvider, which is expected to be a thread-safe component.

## API Surface
The primary public contract of this class is its role as a command, fulfilled by implementing the AbstractAsyncCommand interface. Direct method invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | Network-bound | Asynchronously resolves a username to a UUID via a network call. On success, it delegates removal to the HytaleWhitelistProvider and sends feedback to the caller via the CommandContext. Handles and reports lookup failures. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is instantiated and registered with the command system, which then handles its invocation based on user input. The following example demonstrates how a parent command might register it.

```java
// Example of how this command is registered by a parent system
HytaleWhitelistProvider provider = ...; // Obtained from a service registry
WhitelistCommand parent = new WhitelistCommand();

// The WhitelistRemoveCommand is added as a subcommand
parent.withSubCommand(new WhitelistRemoveCommand(provider));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WhitelistRemoveCommand()` without providing a valid HytaleWhitelistProvider. The class is designed for dependency injection and will not function without its provider.
-   **Direct Invocation:** Never call the `executeAsync` method directly. The command system is responsible for creating the CommandContext and managing the command lifecycle. Bypassing the system can lead to null pointer exceptions and unpredictable behavior.
-   **Blocking on the Future:** Do not block on the returned CompletableFuture from the main server thread (e.g., by calling `future.join()`). This negates the entire purpose of the asynchronous design and will cause the server to hang.

## Data Pipeline
The flow of data for a typical invocation is initiated by a user and involves multiple systems.

> Flow:
> User Input (`/whitelist remove Notch`) -> Command Parser -> **WhitelistRemoveCommand.executeAsync()** -> AuthUtil (External Network Request) -> HytaleWhitelistProvider -> Filesystem Write -> CommandContext -> User Feedback Message

