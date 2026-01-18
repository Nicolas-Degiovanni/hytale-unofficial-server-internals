---
description: Architectural reference for WorldSetDefaultCommand
---

# WorldSetDefaultCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world
**Type:** Transient

## Definition
```java
// Signature
public class WorldSetDefaultCommand extends CommandBase {
```

## Architecture & Concepts
The WorldSetDefaultCommand class is a concrete implementation within the server's command processing system. It serves as a direct, user-facing endpoint for a specific administrative action: changing the server's default world.

Architecturally, this class acts as a translator between a user-issued command and a core server configuration mutation. It does not manage its own state but instead orchestrates interactions between several high-level, singleton services:
- **Command System:** It registers itself with a name and description, defining the command's structure and arguments.
- **Universe:** It queries the global Universe service to validate that the specified world exists before attempting to set it as the default. This prevents the server from entering an invalid state with a non-existent default world.
- **HytaleServer:** It accesses the central server instance to retrieve the configuration object and persist the change.

This class is a terminal component; it initiates a change but does not produce data for further processing in a pipeline. Its primary output is user feedback sent via the CommandSender.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the command registration system during server bootstrap. The system discovers all CommandBase subclasses and creates instances to build the command hierarchy.
- **Scope:** The object instance persists for the entire server session, held as a reference within the central command registry.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless regarding its execution. The single instance field, nameArg, is a final definition of a command argument, initialized at construction and never modified. All state required for execution, such as the target world name and the command sender, is provided via the CommandContext argument in the executeSync method.
- **Thread Safety:** This class is **not thread-safe**. The executeSync method is designed to be invoked by the server's main command processing loop, which is expected to be single-threaded. It performs non-atomic check-then-act logic (checking if a world exists, then setting it) and modifies global state via HytaleServer.get().getConfig(). Concurrent execution could lead to race conditions and inconsistent server configuration.

**WARNING:** Never invoke the executeSync method from multiple threads. The command system guarantees serialized execution.

## API Surface
The public contract is defined by its role as a command. The primary entry point is the executeSync method, which is called by the framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | void | O(1) | Executes the command. Retrieves the world name from the context, validates its existence against the Universe, and updates the server configuration. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically registered and executed by the server's command handler in response to user input. A server administrator would trigger this command from the console or in-game.

The *framework's* interaction pattern is as follows:

```java
// This is a conceptual example of how the command system invokes the method.
// Do not replicate this pattern.

CommandContext context = buildContextForUserInput("/world setdefault MyWorld");
WorldSetDefaultCommand command = commandRegistry.findCommand("world setdefault");
command.executeSync(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new WorldSetDefaultCommand()`. The command system manages the lifecycle of command objects. Manually creating an instance will result in an object that is not registered with the server.
- **Manual Execution:** Do not call the `executeSync` method directly. Bypassing the command system's invocation logic circumvents critical features like permission checks, argument parsing, and context validation, potentially leaving the system in an unstable state.

## Data Pipeline
This command acts as a sink in a user-input data flow. It consumes a command and arguments, and its primary effect is a state change on a global configuration object.

> Flow:
> User Input (`/world setdefault ...`) -> Network Layer -> Command Parser -> **WorldSetDefaultCommand.executeSync** -> Universe (Read-only validation) -> HytaleServer.Config (State mutation) -> CommandSender (Feedback message)

