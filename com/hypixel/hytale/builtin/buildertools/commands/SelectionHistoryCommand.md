---
description: Architectural reference for SelectionHistoryCommand
---

# SelectionHistoryCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class SelectionHistoryCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The SelectionHistoryCommand class is a command-pattern implementation that serves as a direct, user-facing interface to manipulate player-specific data within the Builder Tools feature set. It acts as a terminal node in the server's command processing system, translating a player's chat input into a specific state change.

Architecturally, this class decouples the core command system from the business logic of the BuilderToolsPlugin. The command system is responsible for routing, parsing, and permission checking, while this class is solely responsible for the final mutation of the BuilderToolsUserData component. It retrieves the target player's state via the Entity Component System (ECS) and modifies a single boolean flag, providing a clear and isolated point of interaction.

## Lifecycle & Ownership
- **Creation:** A single instance of SelectionHistoryCommand is instantiated by the server's command registration system when the BuilderToolsPlugin is loaded. It is then registered with the central CommandSystem under the name "selectionHistory".

- **Scope:** The object instance is a long-lived singleton that persists for the entire duration the plugin is active. It does not hold per-player state; its role is to process executions as they occur.

- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the BuilderToolsPlugin is unloaded or the server shuts down, at which point it is removed from the CommandSystem registry.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only instance field, enabledArg, is an immutable definition of a command argument configured during construction. All stateful operations performed by the execute method target external objects, specifically the BuilderToolsUserData associated with the executing player.

- **Thread Safety:** This class is not thread-safe and is not designed to be. Command execution is expected to be serialized and occur on the main server thread or a world-specific thread. The method mutates player component data, which is an operation that must be synchronized by the owning game loop to prevent race conditions within the ECS. Direct, concurrent invocation of the execute method will lead to undefined behavior and data corruption.

## API Surface
The public contract is defined by its base class, AbstractPlayerCommand. The primary interaction point is the execute method, which is invoked by the framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Sets the selection history recording flag for the executing player. Throws argument-related exceptions if input is invalid. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly from code. It is automatically executed by the server's command system in response to player input. The standard interaction is a player typing the command into the game client.

*Player-invoked example:*
```
/selectionHistory true
```

*System-level (hypothetical) invocation:*
```java
// The CommandSystem finds and executes the command based on parsed input.
// This is handled by the engine and should NOT be replicated by developers.
CommandSystem commandSystem = server.getCommandSystem();
commandSystem.dispatch(player, "selectionHistory false");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SelectionHistoryCommand()` in your own code. The command must be registered with the CommandSystem to function correctly.

- **Manual Invocation:** Never call the `execute` method directly. Doing so bypasses the entire command processing pipeline, including critical permission checks, argument parsing, and context setup. This can lead to security vulnerabilities and server instability.

## Data Pipeline
The flow of data for this command is linear, originating from the player and resulting in a state change and a feedback message.

> Flow:
> Player Chat Input -> Server Network Layer -> CommandSystem Parser -> **SelectionHistoryCommand.execute()** -> BuilderToolsPlugin -> BuilderToolsUserData.setRecordSelectionHistory(value) -> CommandContext.sendMessage() -> Server Network Layer -> Player Chat Output

