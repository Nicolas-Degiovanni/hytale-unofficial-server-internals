---
description: Architectural reference for EntityStatsSetToMaxCommand
---

# EntityStatsSetToMaxCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity.stats
**Type:** Singleton

## Definition
```java
// Signature
public class EntityStatsSetToMaxCommand extends AbstractTargetEntityCommand {
```

## Architecture & Concepts
The EntityStatsSetToMaxCommand is a server-side command responsible for mutating a specific statistic on one or more entities, setting its value to the defined maximum. This class operates as a leaf node within the server's broader Command System framework.

Architecturally, it serves as a bridge between user input (a player or system executing a command) and the core game state managed by the Entity-Component-System (ECS). It inherits from AbstractTargetEntityCommand, which abstracts away the complex logic of parsing entity selectors (e.g., @s, @e[type=player]) and resolving them into a concrete list of target entities. This delegation allows EntityStatsSetToMaxCommand to focus solely on its business logic: validating the statistic name and applying the state change.

The command's logic is data-driven, looking up statistic definitions from the EntityStatType asset map. This decouples the command from hard-coded stat names, allowing designers to add or modify entity statistics without requiring code changes to the command itself.

### Lifecycle & Ownership
- **Creation:** A single instance of EntityStatsSetToMaxCommand is instantiated by the server's CommandRegistry during the bootstrap phase. The system discovers and registers all command classes at startup.
- **Scope:** The object is a singleton that persists for the entire server session. It is registered once and reused for every invocation of the "settomax" command.
- **Destruction:** The instance is de-referenced and eligible for garbage collection only when the CommandRegistry is cleared, which typically occurs during a full server shutdown.

## Internal State & Concurrency
- **State:** This class is effectively stateless regarding command execution. Its instance fields, such as entityStatNameArg, are immutable configurations defined in the constructor that describe the command's argument structure. All data related to a specific invocation is passed through method parameters, primarily via the CommandContext object.

- **Thread Safety:** The command instance itself is thread-safe due to its immutable state. However, the execution logic, particularly the static setEntityStatMax method, performs direct mutations on ECS components (EntityStatMap). The Hytale engine's command dispatcher guarantees that the execute method is called on the main server thread, which has exclusive write access to the world state.

**WARNING:** Calling the public static method setEntityStatMax from any thread other than the main world tick thread will result in severe concurrency issues, including data corruption, race conditions, and server instability. All entity state modification must be synchronized with the game loop.

## API Surface
The primary public contract is not a Java method but the command string itself, invoked by a user in-game. The class exposes a public static utility method which contains the core logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setEntityStatMax(context, entities, entityStatName, store) | void | O(N) | **Core Logic.** Iterates through N entities, finds the specified stat, and sets its value to its maximum. Sends feedback messages to the caller via the CommandContext. Throws no exceptions, handling errors via in-game messages. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code by most developers. It is automatically discovered and invoked by the server's command processing system. A user, such as a player or a command block, triggers its execution.

*Example In-Game Invocation:*
```
/entity stats @s health settomax
```
This command targets the executing entity (@s), selects the "health" statistic, and invokes this command to set it to its maximum value.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create instances using `new EntityStatsSetToMaxCommand()`. The object has no utility outside the context of the command registration system.
- **Asynchronous Mutation:** Do not call the static `setEntityStatMax` method from a separate thread. All world state mutations must be performed on the main server thread to ensure data integrity.

## Data Pipeline
The flow for this command begins with user input and ends with a direct mutation of the world's entity data. The command acts as the controller that translates a text-based request into a specific state change within the ECS.

> Flow:
> User Command String -> Server Command Parser -> **EntityStatsSetToMaxCommand.execute** -> EntityStatsModule -> **EntityStatMap.setStatValue** -> World State Change

