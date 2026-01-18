---
description: Architectural reference for BindingsCommand
---

# BindingsCommand

**Package:** com.hypixel.hytale.server.core.io.commands
**Type:** Service Object

## Definition
```java
// Signature
public class BindingsCommand extends CommandBase {
```

## Architecture & Concepts
The BindingsCommand is a concrete implementation of the Command Pattern, designed for server diagnostics and administration. It integrates into the server's command processing system to expose internal network state to operators.

Its sole responsibility is to act as a bridge between the high-level Command System and the low-level `ServerManager`. When invoked, it queries the `ServerManager` for a list of all active network listeners (represented by Netty `Channel` objects) and formats this information into a human-readable message for the command's issuer. This provides a simple, safe mechanism for administrators to verify which network interfaces the server is currently bound to without accessing the underlying infrastructure.

## Lifecycle & Ownership
- **Creation:** An instance of BindingsCommand is created by the server's command registration service during the bootstrap sequence. The system typically scans for all classes extending `CommandBase` and instantiates them for registration.
- **Scope:** The object is a long-lived singleton for the duration of the server's runtime. A single instance is registered and reused for all subsequent invocations of the "bindings" command.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** BindingsCommand is **stateless**. It contains no mutable instance fields and does not cache any data. All information is fetched from the `ServerManager` singleton at the moment of execution. The `MESSAGE_IO_SERVER_MANAGER_BINDINGS` field is a static final constant, representing a translatable string key.

- **Thread Safety:** This class is inherently thread-safe due to its stateless design. However, it is intended to be executed synchronously on the main server thread, as enforced by the `executeSync` method signature. The command system guarantees this execution context, preventing potential race conditions when accessing the `ServerManager`'s internal state. Direct invocation from other threads is not supported and may lead to unpredictable behavior.

## API Surface
The public contract is defined by its superclass, `CommandBase`. The primary entry point is the `executeSync` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | void | O(N) | Executes the command logic. N is the number of active listeners, which is typically a small constant. Throws exceptions if the CommandContext is null. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly by developers in code. It is registered with the command system and invoked by an administrator through a console or remote shell. The system handles the invocation.

A conceptual view of programmatic execution would look like this:
```java
// A developer would not write this; it is handled by the server's CommandManager.
CommandManager commandManager = server.getCommandManager();
CommandContext context = createSomeContextForAnIssuer();

// The manager finds and executes the registered command
commandManager.execute("bindings", context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new BindingsCommand()`. The command will not be registered with the server's command dispatcher and will have no effect. It must be discovered and managed by the engine's command system.

- **Bypassing the Command System:** Avoid calling the `executeSync` method directly. Doing so bypasses critical infrastructure such as permission checks, argument parsing, and thread safety guarantees provided by the `CommandManager`. The `CommandContext` may also be improperly configured, leading to runtime errors.

## Data Pipeline
The data flow for this command is a simple request-response loop for diagnostic information.

> Flow:
> Console Input (`/bindings`) -> CommandManager -> **BindingsCommand.executeSync()** -> ServerManager.getListeners() -> Formatted Message -> CommandContext.sendMessage() -> Console Output

