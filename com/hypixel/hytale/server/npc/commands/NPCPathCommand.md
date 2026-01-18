---
description: Architectural reference for NPCPathCommand
---

# NPCPathCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient

## Definition
```java
// Signature
public class NPCPathCommand extends AbstractCommandCollection {

   public static class PolygonPathCommand extends NPCWorldCommandBase {
      // ...
   }

   public static class SetPathCommand extends NPCWorldCommandBase {
      // ...
   }
}
```

## Architecture & Concepts

The NPCPathCommand class serves as a command-line interface for the server's Non-Player Character (NPC) pathfinding system. It is not a monolithic command but a **Command Collection**, acting as a namespace that groups related sub-commands: `polygon` and the default `set` command. Its primary architectural role is to translate structured text input from a server administrator or script into a concrete `TransientPath` object.

This class forms a critical bridge between the server's Command System and the NPC AI and Entity Component System (ECS). Upon execution, its sub-commands perform the following sequence:

1.  **Argument Parsing:** It decodes and validates user-provided parameters, such as the number of sides for a polygon or a comma-separated string of movement instructions.
2.  **State Retrieval:** It queries the ECS for the target NPCEntity's current world state, specifically its `TransformComponent` (for position) and `HeadRotation` (for initial orientation).
3.  **Path Generation:** It uses the parsed arguments and initial state to construct a sequence of `RelativeWaypointDefinition` objects.
4.  **Path Assignment:** The generated waypoints are compiled into a `TransientPath` and assigned to the NPCEntity's `PathManager`. This action signals the NPC's AI to begin executing the new movement plan.

This command operates exclusively on the server and is a key tool for level designers and administrators to script and test NPC behaviors without direct code modification.

### Lifecycle & Ownership
-   **Creation:** A single instance of NPCPathCommand is instantiated by the server's central `CommandSystem` during the server bootstrap phase. It is then registered into the command hierarchy, typically under the `npc` parent command.
-   **Scope:** The object's lifetime is tied to the server session. It persists in the command registry from server start to server shutdown.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the `CommandSystem` is cleared during server shutdown.

## Internal State & Concurrency
-   **State:** The NPCPathCommand class and its nested sub-commands are fundamentally **stateless**. They hold fields for argument definitions (`RequiredArg`, `OptionalArg`), but these are configuration templates, not mutable state. All runtime data is provided via the `CommandContext` object passed into the `execute` method for each invocation. No state is persisted between command executions.
-   **Thread Safety:** This class is **not thread-safe** and must only be invoked by the main server thread. All interactions with the game `World` and the ECS (`Store`, `Ref`) are fundamentally unsafe to perform from other threads. The `CommandSystem` guarantees that all command executions occur synchronously within the server's main tick loop, preventing race conditions with other game logic.

## API Surface

The public contract is defined by the `execute` methods of its nested classes, which are invoked by the command system. Direct invocation is strongly discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PolygonPathCommand.execute(...) | void | O(N) | Generates a polygonal path with N sides. Fetches entity state from the ECS and assigns a new `TransientPath`. |
| SetPathCommand.execute(...) | void | O(N) | Parses a string of N instruction pairs. Fetches entity state and assigns a new `TransientPath`. Reports parsing errors to the user. |

## Integration Patterns

### Standard Usage

This class is designed to be used via the in-game console or server scripts, not through direct Java calls. The command system routes the text to the appropriate `execute` method.

**Example 1: Creating a 5-sided polygon path with 10-block long sides.**
```
/npc selected path polygon sides:5 length:10
```

**Example 2: Defining a custom path with angle and distance pairs.**
This command instructs the NPC to turn 90 degrees right and move 5 blocks, then proceed forward 10 blocks.
```
/npc selected path "90,5,0,10"
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create an instance using `new NPCPathCommand()`. The class is useless outside the context of the server's command registry, which handles its lifecycle.
-   **Manual Execution:** Never call the `execute` method directly. It depends on a fully-formed `CommandContext` object that is prepared by the command dispatcher. Manually calling it will result in `NullPointerException` or unpredictable behavior due to missing context.
-   **Asynchronous Invocation:** Do not attempt to execute this command's logic from a separate thread. Modifying entity components like `TransformComponent` or assigning paths via the `PathManager` must be done on the main server thread to prevent data corruption and crashes.

## Data Pipeline

The flow of data for this command begins with user input and ends with an updated AI state for an NPC.

> Flow:
> Server Console Input -> Command System Parser -> **NPCPathCommand** (Argument Validation) -> ECS Query for `TransformComponent` -> `TransientPath` Object Instantiation -> `NPCEntity.getPathManager().setTransientPath()` -> NPC AI System (Movement Execution)

