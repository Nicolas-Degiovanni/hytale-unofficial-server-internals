---
description: Architectural reference for Pos1Command
---

# Pos1Command

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class Pos1Command extends AbstractPlayerCommand {
```

## Architecture & Concepts
The Pos1Command class is a server-side command handler responsible for setting the first corner of a player's selection cuboid for the Builder Tools system. It acts as the primary user-facing entry point for defining a selection area via the in-game chat console.

Architecturally, this class serves as a translator between raw player input and the internal state of the Builder Tools system. It adheres to the Command pattern, encapsulating the user's request and its parameters into a single, executable unit.

Its most critical design feature is its deferred execution model. The Pos1Command does not directly manipulate world state or player selection data. Instead, it validates the request and enqueues a lambda function into the BuilderToolsPlugin's processing queue. This decouples the command parsing and validation logic from the core world-editing engine, allowing the plugin to manage task scheduling, batching, and state consistency centrally.

This command can operate in two modes:
1.  **Coordinate-based:** The player provides explicit X, Y, and Z coordinates.
2.  **Position-based:** The player provides no arguments, and the command uses the player's current block position as the target.

## Lifecycle & Ownership
-   **Creation:** A single instance of Pos1Command is instantiated by the server's Command System during the registration phase, typically when the parent BuilderToolsPlugin is loaded. It is not created on a per-player or per-use basis.
-   **Scope:** The object instance is a long-lived singleton that persists for the entire server session or until the owning plugin is disabled. The *execution* of the command, however, is transient and scoped to a single invocation.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down or the BuilderToolsPlugin is unloaded, at which point the Command System clears its command registry.

## Internal State & Concurrency
-   **State:** The Pos1Command class is effectively stateless. Its fields, such as xArg, yArg, and zArg, define the command's argument structure but do not store any state related to a specific execution. All necessary data is provided through the CommandContext argument in the execute method.
-   **Thread Safety:** This class is inherently thread-safe. The singleton instance can be considered immutable after construction. The execute method is invoked by the server's main thread or a dedicated command processing thread. **WARNING:** The enqueued task's execution and its interaction with world state are managed by the BuilderToolsPlugin, which is responsible for ensuring its own thread safety.

## API Surface
The public contract of this class is fulfilled by its registration with the server's Command System. Direct programmatic invocation is not a supported use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | Framework callback. Resolves the target position and enqueues a state-change operation with the BuilderToolsPlugin. |

## Integration Patterns

### Standard Usage
The only intended use of this class is through player interaction in the game client. The server's command parser routes the request to the registered Pos1Command instance.

```
// Player enters one of the following commands in chat:

// Sets position 1 to the player's current location
/pos1

// Sets position 1 to the specified coordinates
/pos1 100 64 -250
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new Pos1Command()`. The command will not be registered with the server and will have no effect. The server framework manages its lifecycle.
-   **Manual Execution:** Do not attempt to call the `execute` method directly. It requires a fully-formed CommandContext and ECS Store references that can only be reliably provided by the command processing system. Bypassing the system breaks permissions checks and context resolution.

## Data Pipeline
The flow of data for a successful command execution follows a clear, decoupled path from user input to world state modification.

> Flow:
> Player Chat Input (`/pos1 ...`) -> Server Network Listener -> Command Parser -> **Pos1Command.execute()** -> BuilderToolsPlugin Task Queue -> Builder Tools State System -> Player Selection Component Update

