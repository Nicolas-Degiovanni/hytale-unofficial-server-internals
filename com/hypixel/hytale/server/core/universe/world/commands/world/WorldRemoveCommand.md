---
description: Architectural reference for WorldRemoveCommand
---

# WorldRemoveCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world
**Type:** Transient

## Definition
```java
// Signature
public class WorldRemoveCommand extends CommandBase {
```

## Architecture & Concepts
The WorldRemoveCommand class is a concrete implementation of the Command Pattern, designed to handle the server-side logic for removing a game world. It is a self-contained unit of execution that encapsulates all validation and state-modification logic required for this specific administrative action.

This command acts as a transactional controller that interfaces between a CommandSender (a player or the console) and the core Universe service. Its primary architectural role is to enforce server integrity rules before committing a destructive change. It prevents operators from putting the server into an invalid state, such as having no worlds loaded or removing the designated default world that players spawn into.

Upon successful validation, it delegates the final removal operation to the Universe singleton, which manages the server's collection of active worlds.

## Lifecycle & Ownership
- **Creation:** A single instance of WorldRemoveCommand is instantiated by the server's command registration system during the initial bootstrap sequence. It is discovered, constructed, and stored in a central command registry.
- **Scope:** The command definition object is a long-lived singleton that persists for the entire server session. It does not hold per-execution state.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless after its constructor runs. The `nameArg` field is configured once upon instantiation and is treated as immutable thereafter. All state required for execution, such as the target world name and the command's sender, is provided externally via the `CommandContext` object.
- **Thread Safety:** **This class is not thread-safe.** The method name `executeSync` is a strict contract indicating that it must be invoked on the main server thread. Executing this command from any other thread will lead to severe concurrency issues, including race conditions and data corruption within the global Universe state. The underlying `Universe.removeWorld` operation is not designed for concurrent access.

## API Surface
The public API is minimal, as the class is designed to be invoked exclusively by the server's command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(1) | Executes the world removal logic. Throws no exceptions, instead sending feedback messages to the CommandSender. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically executed by the command processing system when a user issues the corresponding command in-game or via the server console.

The system handles parsing the input and invoking the command:

```
// User input that triggers this command
/world remove MyTestWorld
/world rm MyTestWorld
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldRemoveCommand()`. The command system manages the lifecycle of this object. Manual instantiation serves no purpose as it will not be registered to handle any input.
- **Manual Invocation:** Directly calling the `executeSync` method is a critical error. This bypasses the command system's entire pipeline, including permission checks, argument validation, and thread management.
- **State Modification:** Do not attempt to modify the internal state of the command object after its construction. It is designed to be configured once and then treated as immutable.

## Data Pipeline
WorldRemoveCommand acts as an endpoint in a control flow rather than a step in a data transformation pipeline. It receives an instruction and orchestrates a state change within other systems.

> Flow:
> CommandSender Input -> Command Parser -> **WorldRemoveCommand.executeSync()** -> Universe.removeWorld() -> CommandSender.sendMessage() -> Network Response<ctrl63>

