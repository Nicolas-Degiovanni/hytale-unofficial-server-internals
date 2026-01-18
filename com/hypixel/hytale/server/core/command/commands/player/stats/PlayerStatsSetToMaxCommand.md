---
description: Architectural reference for PlayerStatsSetToMaxCommand
---

# PlayerStatsSetToMaxCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.stats
**Type:** Transient

## Definition
```java
// Signature
public class PlayerStatsSetToMaxCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PlayerStatsSetToMaxCommand is a specific command handler within the server's command processing system. It serves as a specialized adapter, bridging user-facing command syntax to a more generic, underlying entity manipulation function.

Architecturally, this class embodies the **Composition over Inheritance** principle. While it inherits from AbstractTargetPlayerCommand to integrate with the command system and leverage player-targeting logic, its primary function is to delegate its core task to the static method `EntityStatsSetToMaxCommand.setEntityStatMax`. This design decouples the player-specific command from the generic entity-stat logic, promoting code reuse and simplifying maintenance.

This class is a terminal node in the command dispatch chain. It is responsible for parsing its specific arguments and orchestrating the modification of an entity's state within the game world.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's command registration system during the server bootstrap phase. A single instance is typically created and registered under its command name, "settomax".
- **Scope:** Session-scoped. The object persists for the entire lifetime of the running server.
- **Destruction:** The instance is eligible for garbage collection when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless regarding command execution. Its only instance field, `entityStatNameArg`, is a final definition of a required command argument. It is initialized once in the constructor and is immutable thereafter. All state required for execution is passed into the `execute` method via the CommandContext and other parameters.

- **Thread Safety:** This class is thread-safe. The immutable nature of its internal state and the reliance on method-scoped parameters for execution prevent race conditions within the class itself. However, thread safety of the overall operation is dependent on the atomicity and thread safety of the downstream systems it calls, specifically the `EntityStore` and the `EntityStatsSetToMaxCommand.setEntityStatMax` implementation. The command system typically ensures that command execution is serialized or handled by a dedicated thread.

## API Surface
The public API is defined by its role as a command handler. Direct invocation is not an intended use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | Delegated | Overrides the base class method to implement the command's logic. Parses the stat name and delegates the state modification to a generic entity stat utility. |

## Integration Patterns

### Standard Usage
This class is not used directly by developers. It is invoked by the server's command system in response to user input. A server administrator or a player with sufficient permissions would execute it via the game's console or chat.

**Example User Input:**
```plaintext
/stats somePlayer settomax health
```

The command system parses this input, identifies the target player "somePlayer", resolves the subcommand "settomax" to this class, and invokes its `execute` method with a fully populated CommandContext.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PlayerStatsSetToMaxCommand()`. The command system is solely responsible for the lifecycle of command objects. Manual creation will result in an object that is not registered and will never be called.
- **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses critical infrastructure provided by the command system, including permission checks, argument validation, and context population, which can lead to NullPointerExceptions or unstable server state.

## Data Pipeline
This class acts as a control-flow component that initiates a state change. The data flow is triggered by external input and results in a modification to the world state.

> Flow:
> User Command String -> Server Command Parser -> **PlayerStatsSetToMaxCommand.execute()** -> Argument Retrieval -> `EntityStatsSetToMaxCommand.setEntityStatMax()` -> `EntityStore` Write Operation -> Player Component State Updated

