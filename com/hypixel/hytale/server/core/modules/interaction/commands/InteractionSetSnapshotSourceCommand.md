---
description: Architectural reference for InteractionSetSnapshotSourceCommand
---

# InteractionSetSnapshotSourceCommand

**Package:** com.hypixel.hytale.server.core.modules.interaction.commands
**Type:** Transient

## Definition
```java
// Signature
public class InteractionSetSnapshotSourceCommand extends CommandBase {
```

## Architecture & Concepts
The InteractionSetSnapshotSourceCommand is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central command system. It serves as a terminal node in the command hierarchy, providing a user-facing interface to modify a specific, low-level server configuration.

Its sole responsibility is to parse a user-provided value and use it to mutate the global static state of the `SelectInteraction` class, specifically the `SNAPSHOT_SOURCE` field. This provides a controlled mechanism for operators to change server behavior at runtime without requiring direct code or configuration file access.

Architecturally, this class acts as a bridge between the user input layer (the command system) and a global configuration state. The use of a static field for configuration is a significant design choice, implying that this setting is universally applicable across the entire server instance.

### Lifecycle & Ownership
- **Creation:** A single instance is created by the command registration system during server bootstrap. The system scans the classpath for `CommandBase` subclasses and instantiates them.
- **Scope:** The object instance persists for the entire server session, held as a reference within the central command registry.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only upon server shutdown when the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its internal fields are final and define the command's argument structure. However, it acts as a mutator for the external, static, and mutable state held in `SelectInteraction.SNAPSHOT_SOURCE`. This is a critical distinction; the command itself holds no state, but its *operation* is stateful.

- **Thread Safety:** The class is **not thread-safe**. The `executeSync` method name is a strong convention indicating it must be called from the main server thread. It directly modifies a static variable without any locking mechanism. The server's command dispatcher is responsible for ensuring this method is invoked from the correct, synchronized context to prevent race conditions.

## API Surface
The public API is defined by its parent, `CommandBase`. The key method is the implementation of `executeSync`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | void | O(1) | Executes the command logic. Parses the required argument, mutates the global `SelectInteraction.SNAPSHOT_SOURCE` field, and sends a confirmation message to the originator. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is designed to be invoked exclusively by the server's command processing system in response to user input.

A server operator would trigger this command's logic by typing the corresponding command in-game or via the server console.

```
// Conceptual flow within the command dispatcher
CommandContext context = buildContextFromUserInput("/interaction set PLAYER_VIEW");
CommandBase command = commandRegistry.find("interaction.set");
command.execute(context); // Internally calls executeSync
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new InteractionSetSnapshotSourceCommand()`. The command system manages the lifecycle of command instances. Creating your own instance will result in an object that is not registered and cannot be executed.
- **Direct Invocation:** Never call `executeSync` directly. Bypassing the command system's `execute` method will skip crucial steps like permission validation, context setup, and asynchronous execution management, leading to instability and security vulnerabilities.
- **External State Mutation:** The command's purpose is to modify a global static variable. This is an inherently fragile pattern. Other systems should be extremely cautious if they also modify `SelectInteraction.SNAPSHOT_SOURCE`, as there are no safeguards against race conditions outside of the main server thread.

## Data Pipeline
The flow of data for this command begins with user input and terminates with a change to a global static variable.

> Flow:
> User Command Input -> Network Packet -> Server Command Dispatcher -> Argument Parser -> **InteractionSetSnapshotSourceCommand.executeSync** -> `SelectInteraction.SNAPSHOT_SOURCE` (Static Field Mutation)

