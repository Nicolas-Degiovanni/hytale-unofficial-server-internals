---
description: Architectural reference for ObjectiveCompleteCommand
---

# ObjectiveCompleteCommand

**Package:** com.hypixel.hytale.builtin.adventure.objectives.commands
**Type:** Utility

## Definition
```java
// Signature
public class ObjectiveCompleteCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The **ObjectiveCompleteCommand** class serves as a command dispatcher for administrative actions related to the adventure objective system. It is not a command itself but rather a collection, or namespace, that groups related sub-commands under the primary `/objective complete` path. Its sole architectural purpose is to register and organize the child commands that perform the actual state modification.

The true logic resides within its static inner classes:
*   **CompleteObjectiveCommand**: Force-completes an entire objective for a player.
*   **CompleteTaskCommand**: Force-completes a single, specific task within a player's active objective.
*   **CompleteTaskSetCommand**: Force-completes all currently active tasks for a given objective.

These commands act as a bridge between the server's command processing system and the underlying game state, which is managed by the **ObjectivePlugin** and the Entity Component System (ECS). The core operational pattern involves parsing player and objective identifiers from the command context, retrieving the live **Objective** instance associated with the player, and then invoking methods on that instance to mutate its state.

The private utility method **getObjectiveFromId** encapsulates the critical lookup logic. It translates a string-based objective ID provided by a user into a concrete **Objective** object by querying the target player's active objective UUIDs and cross-referencing them with the global **ObjectiveDataStore**.

### Lifecycle & Ownership
- **Creation:** A single instance of **ObjectiveCompleteCommand** is instantiated by the server's command registration framework during server bootstrap or when the adventure system plugin is loaded. The constructor immediately creates and registers its child command instances.
- **Scope:** This object is application-scoped. It persists for the entire lifetime of the server session.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection upon server shutdown or when its parent plugin is unloaded.

## Internal State & Concurrency
- **State:** This class and its inner command classes are stateless. They contain no mutable fields. All state they operate on, such as player data and objective progress, is passed into the *execute* method via the **CommandContext** and the ECS **Store**. The class holds immutable, static references to pre-translated **Message** objects for sending feedback to the user.

- **Thread Safety:** This component is not thread-safe and must not be invoked from arbitrary threads. The Hytale server architecture guarantees that all command execution occurs on the main server thread, which has safe access to the game state. Any attempt to call the *execute* method from an external thread will lead to race conditions and world corruption.

## API Surface
The primary contract is not a programmatic API but the command structure it registers with the server. The functional API is the protected *execute* method within each inner class, which is invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CompleteObjectiveCommand.execute | protected void | O(N) | Locates and completes all tasks for a specified objective. Complexity is proportional to the number of active objectives on the player. |
| CompleteTaskCommand.execute | protected void | O(N) | Locates a specific objective and completes a single task by its index. Complexity is proportional to the number of active objectives. |
| CompleteTaskSetCommand.execute | protected void | O(N) | Locates an objective and completes its entire current task set. Complexity is proportional to the number of active objectives. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is invoked exclusively through the server's command line interface by an administrator or via a command block.

```text
# To complete a single task for a player
/objective complete task <player> <objectiveId> <taskIndex>

# To complete an entire objective for a player
/objective complete objective <player> <objectiveId>
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ObjectiveCompleteCommand()`. The command system manages its lifecycle. Creating a new instance will have no effect as it will not be registered.
- **Direct Method Invocation:** Do not call the `execute` method on any of the inner command classes directly. This bypasses critical engine components, including argument parsing, permission checks, and context setup, and will result in unpredictable behavior or server instability.

## Data Pipeline
The flow for this component is initiated by user input and results in a direct mutation of the game state.

> Flow:
> Player Chat or Console Input -> Server Command Parser -> **ObjectiveCompleteCommand** (Sub-command routing) -> **CompleteTaskCommand.execute** -> **getObjectiveFromId** (State lookup) -> **Objective.complete** (State mutation) -> ECS **Store** Update -> Player Feedback Message

