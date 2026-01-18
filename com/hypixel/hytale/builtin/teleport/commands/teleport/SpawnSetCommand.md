---
description: Architectural reference for SpawnSetCommand
---

# SpawnSetCommand

**Package:** com.hypixel.hytale.builtin.teleport.commands.teleport
**Type:** Transient

## Definition
```java
// Signature
public class SpawnSetCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The SpawnSetCommand class is a concrete implementation of the Command Pattern, designed to handle the server-side logic for setting the world's global spawn point. It is a single-purpose, short-lived object that encapsulates the logic for parsing command arguments and modifying persistent world configuration.

This command acts as a transactional endpoint for operator input. It interfaces directly with the command system to receive its context, including the command sender and provided arguments. Its primary responsibility is to derive a target position and rotation, construct a new GlobalSpawnProvider, and persist this change to the active WorldConfig. It effectively translates a user-initiated command into a direct mutation of the world's core settings.

The class distinguishes between two primary execution paths:
1.  **Explicit Coordinates:** The operator provides a specific position and optional rotation.
2.  **Implicit Coordinates:** The command is executed by a player without coordinates, in which case the system uses the player's current position and head rotation as the new spawn point.

This dual-path logic makes the command flexible for both precise programmatic setting and convenient in-game administration.

## Lifecycle & Ownership
- **Creation:** An instance of SpawnSetCommand is created by the server's command dispatch system *on-demand* when a user executes the corresponding command string (e.g., `/spawn set`). It is not a persistent service.
- **Scope:** The object's lifecycle is strictly bound to the execution of a single command. It exists only for the duration of the call to its execute method.
- **Destruction:** The instance is eligible for garbage collection immediately after the command execution completes and a result is sent to the user. No external system maintains a long-term reference to it.

## Internal State & Concurrency
- **State:** The internal state of a SpawnSetCommand instance is defined by its argument parsers (positionArg, rotationArg). These fields are final and are initialized in the constructor. The object is effectively immutable after construction and does not cache or retain any state between invocations.
- **Thread Safety:** This class is **not thread-safe** and is not designed to be. Command execution is marshaled onto the main world thread to ensure that modifications to world state, such as updating the WorldConfig, occur in a serialized and safe manner. All interactions with the World and EntityStore are guaranteed by the command system to happen on the correct thread, preventing race conditions.

## API Surface
The primary contract is the protected execute method, which is invoked by the command system framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Executes the spawn set logic. Mutates the WorldConfig. Throws GeneralCommandException if a non-player executes the command without specifying a position. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically discovered and managed by the server's command system. A user, such as a server administrator, interacts with it via the game client or server console.

The conceptual flow within the engine is as follows:

```java
// PSEUDO-CODE: Represents how the command system invokes this class.
// A developer would NOT write this code.

CommandDispatcher dispatcher = server.getCommandDispatcher();
CommandContext context = createFromPlayerInput("/spawn set 100 64 100");

// The dispatcher resolves "/spawn set" to a new SpawnSetCommand instance
// and invokes it with the necessary world context.
dispatcher.execute(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SpawnSetCommand()`. The object is useless without the context provided by the command framework's `execute` method.
- **Stateful Reuse:** Do not retain a reference to this command object for later use. Each command execution must be handled by a fresh instance to ensure isolation and prevent state corruption.
- **Asynchronous Execution:** Do not call the `execute` method from a different thread. All world modifications must be synchronized with the main server tick. The command system guarantees this; bypassing it will lead to world corruption.

## Data Pipeline
SpawnSetCommand acts as a final sink in a user-input control flow. It translates a command into a direct state change within the world configuration.

> Flow:
> User Input (`/spawn set ...`) -> Network Layer -> Command Parser -> **SpawnSetCommand.execute()** -> WorldConfig.setSpawnProvider() -> WorldConfig.markChanged() -> Confirmation Message -> Network Layer -> User Client

