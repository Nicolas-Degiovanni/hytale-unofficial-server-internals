---
description: Architectural reference for RepairFillersCommand
---

# RepairFillersCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Command Handler

## Definition
```java
// Signature
public class RepairFillersCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The RepairFillersCommand class is a concrete implementation of the Command Pattern, designed to integrate with the server's core command processing system. It serves as the player-facing entry point for the `/repairfillers` command, which is exclusive to the Builder Tools feature set.

Architecturally, this class acts as a lightweight **dispatcher and validator**. It does not contain the complex logic for modifying world data. Instead, its primary responsibilities are:
1.  **Registration:** Declaring the command name, description, and required permission level (Creative mode) to the server.
2.  **Context Validation:** Verifying that the command is executed by a valid player in a state suitable for builder tool operations, using PrototypePlayerBuilderToolSettings.
3.  **Delegation:** Enqueuing the actual repair operation to the BuilderToolsPlugin work queue.

This design decouples the user-facing command interface from the underlying implementation. It ensures that the command execution logic remains fast and non-blocking on the main server thread, offloading the potentially intensive world modification task to a managed, asynchronous queue.

## Lifecycle & Ownership
-   **Creation:** A single instance of RepairFillersCommand is instantiated by the server's command registry during the bootstrap sequence. The system discovers and registers all classes extending AbstractCommand or a similar base type.
-   **Scope:** The object is a singleton for the lifetime of the server. This single instance handles all invocations of the `/repairfillers` command from any player.
-   **Destruction:** The instance is de-referenced and eligible for garbage collection only when the server shuts down or the parent BuilderToolsPlugin is unloaded.

## Internal State & Concurrency
-   **State:** The class is effectively immutable after construction. Its state, consisting of the command name and permission group, is set once in the constructor and never changes. The execute method is stateless, operating exclusively on the parameters provided by the command system during invocation.
-   **Thread Safety:** This class is thread-safe by design. The execute method is called by the server's main thread. The most critical operation, `BuilderToolsPlugin.addToQueue`, delegates the task to a system that is responsible for its own concurrency management. This prevents race conditions and ensures that world modifications are processed in a controlled, sequential manner, avoiding direct, unmanaged access to shared world state from the command handler.

## API Surface
The primary contract is the protected `execute` method, which is invoked by the framework, not by developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Validates player state and enqueues a repair task. Sends a confirmation message to the player. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is invoked automatically by the server when a player in Creative mode types the command into the chat.

```
// Player-invoked via chat console
/repairfillers
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new RepairFillersCommand()`. The object is useless unless registered with the server's command system, which is handled automatically at startup.
-   **Manual Invocation:** Do not call the `execute` method directly. Doing so bypasses the entire command processing pipeline, including permission checks and context injection. The required `CommandContext`, `Store`, and `Ref` parameters are managed by the engine and cannot be reliably constructed manually.

## Data Pipeline
The flow for this command is initiated by a player and results in a deferred task execution.

> Flow:
> Player Chat Input (`/repairfillers`) -> Server Command Parser -> **RepairFillersCommand.execute()** -> Pre-condition Check (isOkayToDoCommandsOnSelection) -> BuilderToolsPlugin Work Queue -> World Modification Task -> Player Confirmation Message

