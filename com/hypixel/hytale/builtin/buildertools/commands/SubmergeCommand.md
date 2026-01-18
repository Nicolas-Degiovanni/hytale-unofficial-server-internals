---
description: Architectural reference for SubmergeCommand
---

# SubmergeCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class SubmergeCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The SubmergeCommand class is a server-side command handler within the Builder Tools plugin framework. It serves as a user-facing entry point for a complex world modification operation: filling a player-defined selection with a specified fluid.

Architecturally, this class acts as a lightweight controller. Its primary responsibility is not to perform the world modification itself, but to:
1.  Define the command syntax, including its name (`submerge`), aliases (`flood`), and required arguments (`fluid-item`).
2.  Enforce permissions, restricting its use to players in Creative mode.
3.  Receive and parse player input via the server's central command system.
4.  Perform initial validation on the player's state and the provided arguments.
5.  Translate the validated command into a deferred task and delegate it to the BuilderToolsPlugin's asynchronous processing queue.

This delegation is a critical design choice. By queuing the operation, the server avoids blocking the main game loop on potentially massive world edits, ensuring server performance and stability. The command is merely the trigger; the actual work is handled by a separate, more robust system.

### Lifecycle & Ownership
-   **Creation:** An instance of SubmergeCommand is created and registered by the server's command system during the startup phase of the BuilderToolsPlugin. It is not created per-execution.
-   **Scope:** The object exists as a singleton for the entire lifecycle of the plugin. It persists as long as the server is running and the plugin is enabled.
-   **Destruction:** The instance is garbage collected when the BuilderToolsPlugin is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is effectively stateless regarding command execution. Its only instance field, `fluidItemArg`, defines the command's argument structure and is immutable after construction. All state required for execution (the player, the world, the context) is passed as parameters to the `execute` method.
-   **Thread Safety:** The SubmergeCommand instance is thread-safe. As its internal state is immutable, it can be safely referenced by the command system from any thread. The `execute` method itself is designed to be called by the server's main thread. It safely hands off the world modification logic to the `BuilderToolsPlugin` queue, which is responsible for its own concurrency management.

## API Surface
The public API is minimal and intended for framework-level interaction only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(1) | Validates player state and arguments, then enqueues a world modification task. The complexity is constant time as it only submits a task, but the resulting asynchronous operation is proportional to the volume of the player's selection. |

## Integration Patterns

### Standard Usage
This class is not designed for direct invocation by developers. It is automatically invoked by the server's command processing system when a player executes the command in-game. The framework provides the necessary context and state.

A conceptual view of the framework's interaction:
```java
// PSEUDOCODE: How the server's command system might use this class.
// Developers should NOT write this code.

CommandSystem commandSystem = server.getCommandSystem();
Player player = server.getPlayer("username");
String[] args = new String[]{"hytale:water"};

// The system finds the SubmergeCommand and invokes it with the correct context.
commandSystem.dispatch(player, "submerge", args);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new SubmergeCommand()`. The command must be registered with and managed by the server's command system to function correctly.
-   **Manual Execution:** Calling the `execute` method directly from other plugin code is a severe anti-pattern. This bypasses critical framework infrastructure, including permission checks, argument parsing, and context injection, leading to unpredictable behavior and NullPointerExceptions.

## Data Pipeline
The flow of data for a submerge operation begins with player input and ends with a modification to the world state, with this class acting as an early-stage controller.

> Flow:
> Player Chat Input (`/submerge hytale:water`) -> Server Command Parser -> **SubmergeCommand.execute()** -> Argument Validation (`isFluidItem`) -> Task Creation (Lambda) -> `BuilderToolsPlugin.addToQueue` -> World Edit Worker Thread -> `s.set(pattern)` -> World Storage Modification

