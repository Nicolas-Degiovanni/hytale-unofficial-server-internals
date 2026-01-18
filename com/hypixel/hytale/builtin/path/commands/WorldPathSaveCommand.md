---
description: Architectural reference for WorldPathSaveCommand
---

# WorldPathSaveCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Transient

## Definition
```java
// Signature
public class WorldPathSaveCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldPathSaveCommand class is a concrete implementation of the Command design pattern, specifically tailored for server-side world management. It functions as a terminal endpoint within the server's command processing system, translating a user-issued command into a direct persistence action.

Its primary role is to act as a bridge between the command system and the world's configuration state. When invoked, it locates the active WorldPathConfig associated with the current world context and triggers its serialization to disk. This class does not contain any business logic related to the configuration itself; it is purely a dispatch and invocation mechanism.

This command is a standard part of the administrative toolset, allowing operators to explicitly save pathfinding and navigation configurations for a world without requiring a full world save or server shutdown.

## Lifecycle & Ownership
- **Creation:** A single instance is created by the server's command registration system during the server bootstrap phase. The system scans for classes annotated or inheriting from command bases and instantiates them for its internal registry.
- **Scope:** The singleton instance persists for the entire lifecycle of the server. It is a stateless service object whose methods are invoked on demand.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server is shutting down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. It contains no instance fields that store data between invocations. The static field MESSAGE_UNIVERSE_WORLD_PATH_CONFIG_SAVED is a final, immutable message key, loaded once at class initialization.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. However, the execution context is critical. The command system guarantees that command execution occurs on the main server thread or a world-specific thread, preventing concurrent modification of the World object passed into the execute method. The underlying `save` operation on WorldPathConfig is expected to be a blocking I/O call and must handle its own file access synchronization if necessary.

## API Surface
The public API is defined by its parent, AbstractWorldCommand. The core logic resides in the overridden `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | I/O Bound | Triggers the persistence of the world's pathfinding configuration to disk. Sends a confirmation message to the command issuer via the CommandContext. |

## Integration Patterns

### Standard Usage
This command is not intended to be invoked directly from code. It is registered with the server's command handler and executed by an authorized user, such as a server administrator, via the server console or in-game chat.

*Example User Interaction:*
```
/worldpath save
```
This input is parsed by the command system, which identifies WorldPathSaveCommand as the handler. The system then populates the CommandContext and World arguments and invokes the `execute` method.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldPathSaveCommand()`. The resulting object will not be registered with the command system and will be completely non-functional as a command.
- **Manual Execution:** Avoid calling the `execute` method directly. Doing so bypasses the command system's essential middleware, including permission checks, argument parsing, and context setup. This can lead to inconsistent state or security vulnerabilities.

## Data Pipeline
The data flow for this command is unidirectional, initiating a state change from memory to persistent storage.

> Flow:
> User Input (`/worldpath save`) -> Command Dispatcher -> **WorldPathSaveCommand.execute()** -> World.getWorldPathConfig().save() -> Filesystem I/O
>
> Feedback Flow:
> **WorldPathSaveCommand.execute()** -> CommandContext.sendMessage() -> Network Layer -> User Client

