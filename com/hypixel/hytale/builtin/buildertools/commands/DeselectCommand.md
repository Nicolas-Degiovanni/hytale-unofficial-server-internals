---
description: Architectural reference for DeselectCommand
---

# DeselectCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Command Handler Singleton

## Definition
```java
// Signature
public class DeselectCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The DeselectCommand class encapsulates the server-side logic for the player-facing `/deselect` chat command. It serves as a direct entry point from the server's command system into the specialized Builder Tools plugin.

Architecturally, this class is a classic implementation of the **Command Pattern**. It does not contain the business logic for clearing a selection itself. Instead, its primary responsibility is to:
1.  Verify the command's preconditions, such as player permissions (Creative mode) and the state of the player's current selection.
2.  Translate the player's command invocation into a deferred task.
3.  Delegate the actual work by submitting this task to the central `BuilderToolsPlugin` work queue.

This delegation decouples the command parsing and validation logic from the core game state modification logic. By queuing the action, it ensures that the operation is executed synchronously with the main game loop on a subsequent tick, preventing race conditions and maintaining world state integrity. It operates entirely within the server's Entity-Component-System (ECS) paradigm, targeting the player entity who issued the command.

### Lifecycle & Ownership
-   **Creation:** A single instance of DeselectCommand is instantiated by the server's command registration system when the `BuilderToolsPlugin` is loaded. The command system discovers and registers this handler for the "deselect" and "clearselection" identifiers.
-   **Scope:** The instance is a singleton within the command registry and persists for the entire server session, or until the parent plugin is unloaded.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the `BuilderToolsPlugin` is disabled or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is **stateless**. Its internal fields (command name, alias, permission group) are configured once in the constructor and are effectively immutable. The `execute` method operates exclusively on the arguments provided by the command system, holding no state between invocations.

-   **Thread Safety:** The `execute` method is designed to be called by the server's main command processing thread. It is **not thread-safe** for concurrent invocations. The critical concurrency control mechanism is its hand-off to the `BuilderToolsPlugin` queue, which is responsible for synchronizing the submitted task with the main server tick. This design avoids the need for locks within the command class itself.

## API Surface
The public contract is defined by its constructor for framework registration and the `execute` method for invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DeselectCommand() | constructor | O(1) | Registers the command name, alias, and required permission group. Intended for framework use only. |
| execute(...) | protected void | O(1) | Validates preconditions and queues a deselection task with the BuilderToolsPlugin. The complexity is constant as it only involves a single queue operation. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. The system is designed to be driven by player input through the game's chat console.

> A player with Creative mode permissions types `/deselect` or `/clearselection` into the chat. The server's command dispatcher resolves this input to the registered DeselectCommand instance and invokes its `execute` method with the appropriate context.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new DeselectCommand()`. This creates an orphan object that is not registered with the server's command system and will never be executed.
-   **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses the server's critical permission checks, context injection, and argument parsing pipeline, which can lead to instability or security vulnerabilities.

## Data Pipeline
The flow of data and control for a deselection action is linear and asynchronous, passing through several key systems.

> Flow:
> Player Chat Input -> Server Command Parser -> **DeselectCommand.execute()** -> BuilderToolsPlugin Work Queue -> Server Game Tick -> Builder Tools Selection System -> Player Component State Update

