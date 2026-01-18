---
description: Architectural reference for MountCommand
---

# MountCommand

**Package:** com.hypixel.hytale.builtin.mounts.commands
**Type:** Transient

## Definition
```java
// Signature
public class MountCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The MountCommand class implements the Composite design pattern to serve as a top-level command router within the server's command processing system. It does not contain any execution logic itself; its sole responsibility is to group related sub-commands, such as DismountCommand and MountCheckCommand, under a single, user-facing namespace: `/mount`.

This class acts as a non-terminal node in the server's command tree. When the CommandManager parses a player's input, it first matches the "mount" token and dispatches the remainder of the command to this object. MountCommand then delegates the final parsing and execution to the appropriate registered sub-command. This architectural choice simplifies command registration, centralizes permission management for a group of related functionalities, and improves discoverability for server administrators and players.

## Lifecycle & Ownership
- **Creation:** Instantiated once by the server's central CommandManager or a similar registration service during the server bootstrap sequence. The system likely discovers this class via reflection or a predefined list of built-in commands.
- **Scope:** Application-scoped. The instance persists in the CommandManager's registry for the entire lifetime of the server process.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection only upon server shutdown when the CommandManager clears its command registry.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal list of sub-commands is populated exclusively within the constructor and is not designed to be modified at runtime. The object's state is fixed after its initial creation.
- **Thread Safety:** This class is inherently thread-safe due to its immutable state post-construction. However, the command system as a whole is responsible for ensuring that the *execution* of any command logic is performed on the main server thread to prevent race conditions when modifying game state. Accessing the MountCommand instance from any thread is safe; invoking command execution through it is not.

## API Surface
The primary public contract is the constructor, used during the registration phase. Other methods are inherited from AbstractCommandCollection for internal use by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MountCommand() | Constructor | O(1) | Initializes the command with its name ("mount") and base permission key. Registers its child sub-commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in game logic. It serves as a structural component for the command system. Developers creating new command collections should follow this registration pattern.

```java
// Example of how the command system would register this command
// This code would exist within a central command registry, not game logic.

CommandManager manager = server.getCommandManager();
manager.register(new MountCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not instantiate this class within game logic using `new MountCommand()`. It is designed to be registered once at server startup. Multiple instances would be redundant and wasteful.
- **State Mutation:** Do not attempt to add or remove sub-commands after the object has been constructed. The internal collection is not designed for runtime modification and doing so could lead to unpredictable command routing behavior.

## Data Pipeline
MountCommand acts as a routing step in the server's command processing pipeline. It receives partially processed data and dispatches it to a more specific handler.

> Flow:
> Player Chat Input -> Network Packet -> Server Command Parser -> CommandManager -> **MountCommand** (routes to sub-command) -> DismountCommand / MountCheckCommand -> Game Logic Execution

