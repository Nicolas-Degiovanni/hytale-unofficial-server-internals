---
description: Architectural reference for PermCommand
---

# PermCommand

**Package:** com.hypixel.hytale.server.core.permissions.commands
**Type:** Configuration Object

## Definition
```java
// Signature
public class PermCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PermCommand class serves as a top-level command router within the server's command handling system. It embodies the **Composite Pattern**, acting as a container that aggregates multiple related sub-commands (PermGroupCommand, PermUserCommand, PermTestCommand) under a single, user-facing entry point: `/perm`.

Its sole responsibility is to group and delegate. It contains no business logic itself. When the server's CommandManager processes a command string beginning with "perm", it dispatches the request to this object. PermCommand, through the mechanisms inherited from AbstractCommandCollection, then routes the execution to the appropriate sub-command based on the next argument in the command string (e.g., "group", "user").

This design decouples the main command registry from the specific implementation details of permission management, promoting a clean, hierarchical command structure that is easy to extend and maintain.

### Lifecycle & Ownership
- **Creation:** An instance of PermCommand is created during the server's bootstrap sequence. A central CommandManager or a similar registry service is responsible for instantiating all core command handlers, including this one.
- **Scope:** The object is a long-lived component. It is instantiated once and persists for the entire lifecycle of the server session, remaining registered in the CommandManager.
- **Destruction:** The instance is eligible for garbage collection only upon server shutdown, when the CommandManager that holds a reference to it is destroyed.

## Internal State & Concurrency
- **State:** The state of a PermCommand instance is effectively **immutable** after construction. The command name ("perm") and its collection of sub-commands are defined within the constructor and are not intended to be modified at runtime.
- **Thread Safety:** This class is **not thread-safe**. Command execution in Hytale is expected to occur on the main server thread to prevent data corruption and race conditions with the game world state. All interactions with this class and its sub-commands must be synchronized with the primary server tick loop.

## API Surface
The primary public contract of PermCommand is not defined by its own methods but by its registration with the command system and the behavior inherited from AbstractCommandCollection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PermCommand() | Constructor | O(1) | Instantiates the command, setting its name and registering its child sub-commands. |

**Warning:** The functional API (e.g., methods for execution and tab-completion) is defined in the parent AbstractCommandCollection and is intended for internal use by the CommandManager only.

## Integration Patterns

### Standard Usage
Developers do not interact with instances of this class directly. The standard pattern is to instantiate and register it with the server's central command authority during initialization.

```java
// Example from a hypothetical CommandRegistry service
// This code runs once at server startup.

CommandManager commandManager = server.getCommandManager();
commandManager.register(new PermCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call methods on a PermCommand instance directly. All command execution must flow through the server's CommandManager to ensure proper lifecycle, permission checks, and thread safety.
- **Runtime Modification:** Do not attempt to add or remove sub-commands after the object has been constructed. The command collection is not designed to be dynamically altered post-initialization.

## Data Pipeline
PermCommand acts as a routing step in the data flow from player input to system execution. It validates the primary command token and delegates the payload to the next component in the chain.

> Flow:
> Player Input (`/perm user HytaleDev setgroup admin`) -> Network Packet -> Server Command Dispatcher -> **PermCommand** (Identifies "perm", delegates to sub-command) -> PermUserCommand (Parses "HytaleDev setgroup admin") -> PermissionService -> Database/Configuration Update -> Command Result -> Network Response -> Player Feedback

