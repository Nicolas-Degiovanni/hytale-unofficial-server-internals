---
description: Architectural reference for SetCommand
---

# SetCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class SetCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The SetCommand class is a server-side command handler responsible for processing the player-facing `/set` command. It serves as the primary entry point for one of the most fundamental operations within the Builder Tools plugin: filling a selected region with a specified block pattern.

Architecturally, this class acts as a translator and a delegator. It translates the raw text input from a player into a structured, deferred world modification operation. It does not perform the block placement itself. Instead, it validates the command arguments and the player's state, then enqueues a task with the BuilderToolsPlugin.

This decoupling is a critical design choice. It prevents the command processing thread from blocking on potentially massive, time-consuming world edits, which could otherwise freeze the server. The SetCommand's sole responsibility is to parse, validate, and hand off the work, ensuring the server's main loop remains responsive. It leverages the core server's command system for argument parsing (ArgTypes.BLOCK_PATTERN) and permission enforcement.

### Lifecycle & Ownership
- **Creation:** A single instance of SetCommand is created by the server's command registry during the loading phase of the BuilderToolsPlugin. The command system discovers and registers all classes that extend command base classes like AbstractPlayerCommand.
- **Scope:** The SetCommand object is a long-lived singleton, persisting for the entire duration the BuilderToolsPlugin is active. However, its `execute` method is invoked transactionally for each command usage, with a new, ephemeral CommandContext.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the BuilderToolsPlugin is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** The class holds one significant piece of internal state: the `patternArg` field. This field defines the structure and requirements of the command's arguments. It is initialized once in the constructor and is treated as immutable thereafter. The class is otherwise stateless regarding command execution; all necessary data (player, world, arguments) is passed into the `execute` method.
- **Thread Safety:** This class is thread-safe. The `execute` method is designed to be called by the server's main command processing thread. Its internal state is read-only after construction. Crucially, it achieves concurrency safety by offloading the stateful, blocking work of world modification to the BuilderToolsPlugin's internal queue. This queue is responsible for managing its own concurrency and executing world edits in a safe, sequential, or parallel manner, away from the command-handling thread.

## API Surface
The primary public contract is the `execute` method, which is an override from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Parses arguments and enqueues a world modification task. The complexity of the enqueued task is proportional to the volume of the player's selection. Throws no exceptions directly but relies on the command system for argument validation. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic invocation. It is automatically discovered and executed by the server's command system when a player with appropriate permissions enters the command in chat.

A typical user interaction flow:
1. A player with creative mode and `hytale.editor.selection.modify` permission makes a selection in the world.
2. The player types the command into chat: `/set hytale:stone`
3. The server's command handler routes this input to the registered SetCommand instance.
4. The `execute` method is called, which queues the fill operation.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create an instance using `new SetCommand()`. It will not be registered with the server's command system and will have no effect.
- **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses the server's entire command processing pipeline, including permission checks, argument parsing from raw strings, and context setup. This can lead to unpredictable behavior and server instability.

## Data Pipeline
The SetCommand acts as a specific step in a larger data flow that begins with player input and ends with a modified world state.

> Flow:
> Player Chat Input (`/set ...`) -> Server Command Parser -> **SetCommand.execute()** -> BuilderToolsPlugin Task Queue -> World Edit Operation -> World Storage Modification

