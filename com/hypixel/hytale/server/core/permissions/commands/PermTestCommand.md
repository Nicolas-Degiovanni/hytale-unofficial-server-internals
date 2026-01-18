---
description: Architectural reference for PermTestCommand
---

# PermTestCommand

**Package:** com.hypixel.hytale.server.core.permissions.commands
**Type:** Transient

## Definition
```java
// Signature
public class PermTestCommand extends CommandBase {
```

## Architecture & Concepts
The PermTestCommand class is a concrete implementation of the Command Pattern, designed specifically for server-side execution. It functions as a diagnostic tool for administrators, providing a direct interface to query the server's permission system.

As a subclass of CommandBase, it integrates seamlessly into the server's command processing framework. The framework is responsible for parsing the raw command input, mapping it to this class, validating arguments against the defined schema (nodesArg), and invoking the execution logic within a controlled context. This class's sole responsibility is to perform the permission check and format the response; it does not contain any logic related to command registration or parsing.

It acts as a bridge between a user-level action (a typed command) and a core server subsystem (the permission handler associated with a CommandSender).

### Lifecycle & Ownership
- **Creation:** An instance of PermTestCommand is created by the server's command registration service during the bootstrap phase. This process typically involves reflection or a service locator scanning for all classes that extend CommandBase.
- **Scope:** The object instance persists for the entire server session, held as a reference within the central CommandRegistry. It is not re-instantiated per command execution.
- **Destruction:** The object is marked for garbage collection when the CommandRegistry is cleared, which typically occurs during server shutdown or a comprehensive plugin reload.

## Internal State & Concurrency
- **State:** The class contains minimal state, primarily the `nodesArg` field which defines its argument requirements. This state is configured once in the constructor and is effectively immutable for the object's lifetime. The class does not cache results or maintain state across multiple executions.
- **Thread Safety:** The `executeSync` method, by convention and framework design, is invoked on the main server thread. The framework guarantees that command execution is serialized, preventing concurrent invocations of this method on the same instance. Therefore, the implementation is not required to be thread-safe and contains no internal locking mechanisms.

**WARNING:** Calling `executeSync` from an asynchronous task or a different thread will bypass the framework's safety guarantees and likely lead to race conditions or exceptions within other server systems.

## API Surface
The public contract is primarily defined by its constructor for instantiation by the framework and the overridden `executeSync` method for execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PermTestCommand() | Constructor | O(1) | Registers the command name, description, and argument definitions with the superclass. |
| executeSync(context) | void | O(N) | Executes the permission check logic. Complexity is linear based on N, the number of permission nodes provided. |

## Integration Patterns

### Standard Usage
This class is not intended to be used programmatically by other services. It is designed to be invoked by an authorized user (e.g., a player or console) via the server's command line interface.

The user provides the command name followed by a space-separated list of permission nodes to check.

*Example In-Game Command:*
```
/test hytale.world.edit hytale.entity.spawn
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PermTestCommand()` for any purpose other than registration with the command framework. A standalone instance is non-functional as it is not part of the command dispatch system.
- **Direct Invocation:** Do not call the `executeSync` method directly. This bypasses critical framework steps, including argument parsing, permission checks for the command itself, and context setup. All command execution must flow through the central command dispatcher.

## Data Pipeline
The flow of data for a typical invocation is linear and synchronous, managed entirely by the command framework.

> Flow:
> Raw User Input (`/test ...`) -> Server Network Listener -> Command Dispatcher -> Argument Parser -> **PermTestCommand.executeSync** -> CommandSender.hasPermission -> Message Builder -> CommandContext.sendMessage -> Server Network Emitter -> Client UI

---

