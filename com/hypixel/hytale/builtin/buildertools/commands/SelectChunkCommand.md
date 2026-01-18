---
description: Architectural reference for SelectChunkCommand
---

# SelectChunkCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class SelectChunkCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The SelectChunkCommand is a server-side, player-executable command that integrates the core server command system with the Builder Tools feature set. As a subclass of AbstractPlayerCommand, it adheres to a strict contract for handling commands initiated by a player entity, ensuring that necessary context such as the player's identity, the world state, and entity data stores are correctly injected at runtime.

Its primary architectural role is to act as a user-facing entry point for a complex backend operation. It translates a simple user intent—selecting the current chunk—into a precise, volumetric selection. The command itself does not contain the selection logic; instead, it calculates the chunk boundaries based on the player's current TransformComponent and delegates the actual selection operation to the BuilderToolsPlugin.

This delegation is performed by adding a lambda function to a queue managed by BuilderToolsPlugin. This pattern decouples the command's execution from the world modification, allowing the Builder Tools system to manage and potentially batch selection updates, preventing command execution from blocking the main server thread.

## Lifecycle & Ownership
- **Creation:** A single instance of SelectChunkCommand is instantiated by the server's command registration system when the BuilderToolsPlugin is loaded during server startup. It is registered against the command name "selectchunk".
- **Scope:** The command object is a singleton that persists for the entire server session. Its internal state is non-existent, so each execution is an independent, transient operation.
- **Destruction:** The object is dereferenced and eligible for garbage collection when the server shuts down or the associated plugin is unloaded.

## Internal State & Concurrency
- **State:** The SelectChunkCommand class is entirely stateless. It contains no instance fields and relies solely on the parameters passed to its execute method. This design makes it highly predictable and robust.
- **Thread Safety:** This class is inherently thread-safe due to its stateless nature. The server's command system invokes the execute method from a controlled thread. The subsequent operation is handed off to the BuilderToolsPlugin queue, which is responsible for its own thread management and synchronization when modifying player selection state. Direct concurrent invocation of the execute method is not a supported or possible scenario under the engine's command processing model.

## API Surface
The public API is minimal, consisting of the constructor for framework instantiation and the overridden execute method which forms its primary contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SelectChunkCommand() | constructor | O(1) | Initializes the command with its name, description, and required permission group (Creative). |
| execute(context, store, ref, playerRef, world) | void | O(1) | Calculates chunk boundaries from the player's position and queues a selection update with the BuilderToolsPlugin. |

## Integration Patterns

### Standard Usage
This command is not intended to be invoked programmatically. A user with Creative game mode permissions triggers it by typing the command into the game client's chat console.

> Player Input: `/selectchunk`

The server command system identifies the player, verifies permissions, and invokes the execute method on the registered SelectChunkCommand instance, providing all necessary world and entity context.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SelectChunkCommand()`. The command must be registered with and managed by the server's command system to function correctly.
- **Manual Execution:** Do not call the `execute` method directly. Doing so bypasses the framework's critical permission checks and dependency injection of the `store`, `ref`, and `world` parameters, which will lead to NullPointerExceptions and server instability.
- **State Assumption:** Do not modify this class to hold state. The command system reuses a single instance for all players; any stored state would create severe race conditions and incorrect behavior.

## Data Pipeline
The command initiates a one-way data flow that translates a player's position into a persistent selection state.

> Flow:
> Player Chat Input -> Server Command Parser -> Permission Check -> **SelectChunkCommand.execute()** -> Read Player TransformComponent -> Calculate Chunk Volume -> Enqueue Selection Lambda in BuilderToolsPlugin -> Deferred Execution -> Update Player's PrototypePlayerBuilderToolSettings Component

