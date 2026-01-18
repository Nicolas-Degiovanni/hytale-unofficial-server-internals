---
description: Architectural reference for InstanceEditListCommand
---

# InstanceEditListCommand

**Package:** com.hypixel.hytale.builtin.instances.command
**Type:** Transient

## Definition
```java
// Signature
public class InstanceEditListCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The InstanceEditListCommand is a concrete implementation of the **Command Pattern**, designed to integrate seamlessly with the server's command processing framework. It represents a specific, user-executable action: listing available instance assets.

Architecturally, this class serves as a thin adapter layer. It connects the user-facing command system to the underlying `InstancesPlugin` service. Its sole responsibility is to receive a command execution request, retrieve data from the `InstancesPlugin`, format it into a user-readable message, and send it back to the command's originator.

By extending `AbstractAsyncCommand`, this class adheres to the server's non-blocking command execution contract. This ensures that even if data retrieval were to become a slow operation (e.g., reading from a database or disk), it would not block the main server thread. In this specific implementation, the operation is trivial, but the adherence to the asynchronous pattern is critical for system stability and scalability.

## Lifecycle & Ownership
- **Creation:** An instance of InstanceEditListCommand is created by the server's command registration system when the `InstancesPlugin` is loaded. The framework discovers this class and instantiates it to register the "list" subcommand.
- **Scope:** The object instance persists for the entire server session, held within the central command registry. It is effectively a singleton in the context of the command map, ready to be executed on demand.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down or the parent plugin is unloaded, at which point the command is removed from the registry.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable instance fields and does not cache data between executions. All necessary information is either provided by the `CommandContext` argument or fetched from external services (`InstancesPlugin`) during the `executeAsync` call.

- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. The `executeAsync` method is designed to be called from the server's command executor thread pool. It is critical that any services it calls, such as `InstancesPlugin.get()`, are also thread-safe. The design assumes that multiple commands may be executing concurrently on different threads.

## API Surface
The public API is minimal and intended for framework use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | O(N) | Fetches and sends a list of N instance assets to the command source. The operation is non-blocking. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers. It is invoked automatically by the server's command dispatcher in response to user input.

A user or system would trigger this command via the server console or in-game chat:
```
/instance edit list
```
The framework then routes this input to the `executeAsync` method of the registered InstanceEditListCommand instance.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new InstanceEditListCommand()`. This creates an orphan object that is not registered with the server's command system and will never be executed.
- **Manual Invocation:** Avoid calling the `executeAsync` method directly. Doing so bypasses the framework's essential middleware, including permission checks, argument parsing, and cooldown management. Always dispatch commands through the appropriate server API.

## Data Pipeline
The flow of data for a typical execution is linear and unidirectional, originating from user input and terminating in a message sent back to the user.

> Flow:
> User Input (`/instance edit list`) -> Server Command Parser -> Command Dispatcher -> **InstanceEditListCommand.executeAsync()** -> `InstancesPlugin` -> Raw String List -> `MessageFormat` -> Formatted `Message` -> `CommandContext` -> Network Layer -> Client UI

