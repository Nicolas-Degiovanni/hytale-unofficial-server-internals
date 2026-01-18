---
description: Architectural reference for PlayerStatsAddCommand
---

# PlayerStatsAddCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.stats
**Type:** Transient Command Object

## Definition
```java
// Signature
public class PlayerStatsAddCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PlayerStatsAddCommand class implements a server-side console command for modifying a specific player's statistical attributes. It is a component within the server's Command System, designed to be discovered and executed by a central command processor.

Architecturally, this class serves as a specialized **Facade** or **Adapter**. It provides a user-friendly, player-centric command interface (`/player stats add ...`) but delegates its core logic to the more generic EntityStatsAddCommand. This design pattern promotes code reuse and separates concerns: this class is responsible for parsing player-specific targets and arguments, while the underlying system handles the generic logic of modifying entity component data.

It inherits from AbstractTargetPlayerCommand, which provides the foundational logic for resolving a player name or selector into a specific PlayerRef and its corresponding EntityStore.

### Lifecycle & Ownership
- **Creation:** A single instance is instantiated by the Command System during server bootstrap. The system scans for command classes and registers them in a central command registry.
- **Scope:** The object instance is a long-lived singleton, persisting for the entire server session. It does not hold state related to any single execution.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless**. Its fields, entityStatNameArg and statAmountArg, are final references to argument definitions. They are configured once in the constructor and are immutable thereafter. All state required for an operation is passed into the execute method via the CommandContext parameter.

- **Thread Safety:** The object instance is inherently thread-safe due to its immutable internal state. However, the execution of the command is **not guaranteed to be thread-safe** by the class itself. Command execution involves mutation of shared game state (World, EntityStore). The Command System is responsible for ensuring that command execution is serialized, typically by dispatching all commands to the main server thread to prevent race conditions.

## API Surface
The public contract is defined by its constructor for registration and the overridden execute method for invocation by the Command System.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, sourceRef, ref, playerRef, world, store) | void | O(1) | Executes the command logic. Parses arguments from the context and delegates to EntityStatsAddCommand. Throws argument-related exceptions if parsing fails. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers in game logic. It is designed to be registered with and invoked by the Command System. The system is responsible for parsing user input, resolving the target player, and populating the CommandContext.

A conceptual view of the system's invocation:

```java
// PSEUDOCODE: Represents the Command System's internal dispatch loop
CommandContext context = commandParser.buildContextFromInput("/player stats add p1 health 10");
PlayerStatsAddCommand command = commandRegistry.findCommand("player stats add");

// The system resolves the target player and invokes the command
Ref<EntityStore> targetEntity = resolvePlayerTarget(context);
command.execute(context, /* ... other resolved parameters ... */, targetEntity);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PlayerStatsAddCommand()` in your code. The command will not be registered with the server's command parser and will be unusable.
- **Manual Invocation:** Avoid calling the execute method directly. The CommandContext and other parameters are complex to construct and must be correctly populated by the Command System to ensure valid game state modification.
- **Adding State:** Do not add mutable instance fields to this class. Doing so would break the stateless pattern, making the command non-reentrant and introducing severe thread-safety issues in a multiplayer environment.

## Data Pipeline
The flow of data for a typical command execution follows a clear path from user input to game state modification. This class acts as a specific handler within that pipeline.

> Flow:
> Raw Command String -> Command System Parser -> **PlayerStatsAddCommand** -> Argument Resolution -> Delegation to EntityStatsAddCommand -> EntityStore Component Mutation -> Game State Update

