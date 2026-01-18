---
description: Architectural reference for PrefabPathAddCommand
---

# PrefabPathAddCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabPathAddCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PrefabPathAddCommand class encapsulates the server-side logic for a player-invoked command that adds a new point, or marker, to a predefined NPC path. It serves as a direct interface between the player's input and the world's persistent pathing data.

This class is a component of the server's Command System and follows the Command Pattern. Its primary responsibility is to parse arguments, validate the player's state, and orchestrate modifications to the underlying path data structures. It integrates with two key systems:

1.  **BuilderToolsPlugin:** It queries this plugin to retrieve the player's session-specific state, specifically which NPC path they have currently selected for editing. This decouples the command from direct player state management.
2.  **WorldPathData:** This is a world-level resource that acts as the single source of truth for all NPC paths. The command retrieves the path object from this resource and triggers the modification.

The command itself is stateless between executions; all necessary context is provided via the arguments to the `execute` method.

## Lifecycle & Ownership
-   **Creation:** A single instance of PrefabPathAddCommand is instantiated by the server's command registration system during server startup. It is discovered and registered as an available command handler.
-   **Scope:** The command handler object is a long-lived singleton that persists for the entire server session. It is held in memory by the central command registry.
-   **Destruction:** The instance is dereferenced and garbage collected during server shutdown when the command registry is cleared.

## Internal State & Concurrency
-   **State:** The class contains immutable configuration state defined in its constructor. The fields `pauseTimeArg`, `observationAngleArg`, and `indexArg` are argument definitions that describe the command's parameters to the command system. They do not change after instantiation. The `execute` method operates on state passed into it but does not modify the internal state of the PrefabPathAddCommand instance itself.
-   **Thread Safety:** **This class is not thread-safe.** The `execute` method directly accesses and modifies the ECS `Store`. Such operations must be performed on the main server world thread to prevent race conditions and data corruption. The command system is responsible for ensuring that `execute` is invoked from the correct thread, and developers must not call it from asynchronous tasks.

## API Surface
The public API is designed for consumption by the command system, not for direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | Executes the command logic. Adds a new marker to the active path. Complexity is O(N) where N is the number of existing markers, due to potential list insertion. Throws GeneralCommandException if the player has no active path. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is automatically invoked by the server when a player executes the corresponding chat command.

The conceptual usage is through player input:
```
/path add [pauseTime] [observationAngleDegrees] [index]
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PrefabPathAddCommand()`. The command framework manages the lifecycle of command objects.
-   **Manual Execution:** Never call the `execute` method directly. Doing so bypasses critical infrastructure, including argument parsing, permission checks, and thread safety guarantees provided by the server's command dispatcher. This can lead to severe state corruption.

## Data Pipeline
The flow of data for this command begins with the player and results in a modification of world state.

> Flow:
> Player Chat Input -> Server Command Parser -> **PrefabPathAddCommand.execute()** -> BuilderToolsPlugin (Read Player State) -> WorldPathData (Read World State) -> PrefabPathHelper.addMarker() (Write World State) -> CommandContext (Send Feedback Message to Player)

