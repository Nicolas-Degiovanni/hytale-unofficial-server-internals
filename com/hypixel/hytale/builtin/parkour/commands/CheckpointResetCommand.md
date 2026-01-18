---
description: Architectural reference for CheckpointResetCommand
---

# CheckpointResetCommand

**Package:** com.hypixel.hytale.builtin.parkour.commands
**Type:** Transient

## Definition
```java
// Signature
public class CheckpointResetCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The CheckpointResetCommand is a concrete implementation of the Command Pattern, designed to integrate with the server's central command processing system. It acts as a thin, stateless adapter that bridges player input to the core logic of the ParkourPlugin.

Its primary architectural function is to decouple the command system from the gameplay systems. The server's command dispatcher does not need to understand the mechanics of parkour; it only needs to know how to invoke the standard execute method on an AbstractPlayerCommand. This class translates a specific player-invoked command into a direct, context-aware call to the appropriate service, in this case, the ParkourPlugin.

As a subclass of AbstractPlayerCommand, it is guaranteed to be executed within the context of a specific player entity, providing safe access to player-specific data and the world they inhabit.

## Lifecycle & Ownership
- **Creation:** A single instance of CheckpointResetCommand is instantiated by its parent, ParkourPlugin, during the plugin's initialization phase. It is then registered with the server's global CommandSystem, binding it to the command string "reset" within its parent command's namespace.
- **Scope:** The command object itself is a singleton managed by the CommandSystem. It persists for the entire lifecycle of the ParkourPlugin, which typically aligns with the server's uptime.
- **Destruction:** The instance is garbage collected only when the ParkourPlugin is unloaded and de-registers its commands from the CommandSystem, or during a full server shutdown.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields that store data between invocations. The only field, MESSAGE_COMMANDS_CHECKPOINT_RESET_SUCCESS, is a static final constant, ensuring its value cannot change at runtime. All required context for execution is provided as method arguments by the command system.

- **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless design, concurrent invocations of the execute method would not interfere with each other. However, the server's command system typically serializes command execution for a given player onto a primary game thread to prevent race conditions within downstream systems. The responsibility for thread safety in the game state modification lies with the ParkourPlugin.

## API Surface
The public contract is defined by its constructor and the overridden execute method. Direct invocation is strongly discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CheckpointResetCommand() | constructor | O(1) | Creates the command instance. Intended for framework use only. |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the reset logic. Delegates to ParkourPlugin and sends a confirmation message. Throws assertion errors if the player entity context is invalid. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly in code. It is registered with the server's command framework and invoked by a player typing the corresponding command into the chat console.

The framework is responsible for parsing the command, identifying the correct command object, verifying permissions, and invoking the execute method with the appropriate context.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CheckpointResetCommand()`. The command system requires a single, registered instance to function correctly. Creating new instances will have no effect and will not be reachable by players.
- **Manual Invocation:** Avoid calling the `execute` method directly. Bypassing the command system's dispatcher means you skip critical infrastructure, including permission checks, context validation, and potential command throttling. All command execution must flow through the central dispatcher.

## Data Pipeline
The execution of this command initiates two simple data flows: one for the state change and one for the user feedback.

> **State Change Flow:**
> Player Input (`/checkpoint reset`) → Server Command Dispatcher → **CheckpointResetCommand.execute()** → ParkourPlugin.resetPlayer(uuid) → Player State Mutation

> **Feedback Flow:**
> **CheckpointResetCommand.execute()** → CommandContext.sendMessage() → Server Network Layer → Client Chat UI Update

