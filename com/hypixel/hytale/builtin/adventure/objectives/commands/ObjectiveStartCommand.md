---
description: Architectural reference for ObjectiveStartCommand
---

# ObjectiveStartCommand

**Package:** com.hypixel.hytale.builtin.adventure.objectives.commands
**Type:** Component

## Definition
```java
// Signature
public class ObjectiveStartCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The ObjectiveStartCommand class serves as a command-line interface to the server's objective system. It is not a standalone service but rather a registered component within the server's command processing framework. Its primary architectural role is to act as a bridge, translating operator or script-initiated commands into direct invocations of the core objective management logic, which resides in the ObjectivePlugin.

This class follows the **Command Collection** pattern, acting as a container and namespace for a group of related sub-commands. It does not contain any execution logic itself. Instead, it registers two nested static classes, StartObjectiveCommand and StartObjectiveLineCommand, which correspond to the `objective` and `line` sub-commands respectively.

This design effectively decouples the command parsing and user-facing interface from the underlying game logic. The command's responsibilities are strictly limited to:
1.  Defining the command syntax and arguments (`/objective start objective <id>`).
2.  Parsing and retrieving arguments from the command context.
3.  Performing initial input validation by checking for the existence of the specified ObjectiveAsset or ObjectiveLineAsset.
4.  Delegating the core action of starting an objective to the authoritative ObjectivePlugin.
5.  Providing user-friendly feedback, including "did you mean" suggestions for typos, by leveraging StringUtil for fuzzy string matching against available asset IDs.

## Lifecycle & Ownership
-   **Creation:** An instance of ObjectiveStartCommand is created once by the server's command registration system during the server bootstrap phase or when the Adventure mode plugin is loaded. The constructor immediately registers its sub-commands.
-   **Scope:** The singleton instance persists for the entire server session. It is a stateless component whose methods are invoked by the command dispatcher.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down or the parent plugin is unloaded, at which point its registered commands are removed from the central command registry.

## Internal State & Concurrency
-   **State:** This class and its nested command handlers are fundamentally **stateless**. They do not store or cache any data between executions. All necessary information, such as the target player, world, and entity data, is provided via the `execute` method's parameters for the duration of a single command invocation.
-   **Thread Safety:** Command execution is managed by the server's core engine and is expected to occur on the main world thread. As such, the class is not designed for concurrent access from multiple threads and contains no explicit synchronization mechanisms.

    **Warning:** Any system this command interacts with, such as the ObjectivePlugin or the EntityStore, is responsible for its own thread safety. This command assumes all downstream calls are safe to make from the main server thread.

## API Surface
The primary API is the command registered with the server, not a programmatic Java API. The `execute` method is the contract fulfilled for the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| StartObjectiveCommand.execute(...) | void | O(1) | Validates and initiates an objective via the ObjectivePlugin. Complexity is O(N) on failure for fuzzy-search suggestions. |
| StartObjectiveLineCommand.execute(...) | void | O(1) | Validates and initiates an objective line via the ObjectivePlugin. Complexity is O(N) on failure for fuzzy-search suggestions. |

## Integration Patterns

### Standard Usage
This component is designed to be used exclusively through the server command line or a command block. It is the designated administrative entry point for manually triggering objectives.

```sh
# Starts the full objective with the ID 'main_quest_01' for the executing player.
/objective start objective main_quest_01

# Starts the specific objective line with the ID 'gather_wood_intro' for the executing player.
/objective start line gather_wood_intro
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ObjectiveStartCommand()`. The class is useless outside the context of the server's command registration system. It provides no functionality if instantiated manually.
-   **Direct Method Invocation:** Do not attempt to get a reference to this command and call its `execute` method directly. This bypasses the entire command processing pipeline, including argument parsing, permission checks, and context setup, leading to unpredictable behavior and likely NullPointerExceptions.

## Data Pipeline
The flow for this command is initiated by an external user action and results in a change to the world's state.

> Flow:
> Player Chat Input or Command Block -> Server Command Parser -> Command Dispatcher -> **ObjectiveStartCommand** -> Asset Map (Validation) -> ObjectivePlugin -> World State & EntityStore Update -> Player Notification

