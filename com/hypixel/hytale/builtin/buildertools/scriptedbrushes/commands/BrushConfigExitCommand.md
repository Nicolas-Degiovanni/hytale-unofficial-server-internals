---
description: Architectural reference for BrushConfigExitCommand
---

# BrushConfigExitCommand

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.commands
**Type:** Command

## Definition
```java
// Signature
public class BrushConfigExitCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The BrushConfigExitCommand class is a concrete implementation of the Command Pattern, designed to integrate with the server's core command processing system. By extending AbstractPlayerCommand, it registers itself as a player-executable command, specifically for the "exit" keyword.

Its primary architectural role is to act as a thin bridge between the high-level command system and the specialized state machine of the builder tools. It does not contain any business logic itself. Instead, it translates a player's command input into a direct invocation on the player's specific tool settings, delegating the responsibility of handling the "exit" state transition to the BrushConfigCommandExecutor.

This design decouples the command input mechanism from the builder tool's internal logic, allowing the tool's state management to evolve independently of how it is triggered.

### Lifecycle & Ownership
- **Creation:** A single instance is created by the server's command registration service during the server bootstrap sequence or when the builder tools module is loaded. The system discovers this class and instantiates it to register the "exit" command.
- **Scope:** The instance is a long-lived singleton for the duration of the server's runtime. It persists as long as the command is registered and available.
- **Destruction:** The object is marked for garbage collection when the server shuts down or the associated module is unloaded, at which point the command is de-registered from the command system.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. The command name and description are set once in the constructor and are never modified. The execute method operates exclusively on arguments provided by the command system and does not modify any internal fields.
- **Thread Safety:** The class is inherently **thread-safe**. As a stateless object, concurrent invocations of the execute method will not interfere with each other. The Hytale server's command system likely serializes commands from a single player, but the class itself imposes no concurrency constraints. Thread safety concerns are deferred to the components it calls, such as ToolOperation and BrushConfigCommandExecutor.

## API Surface
The public API is defined by its role as a command and consists solely of the overridden execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Executes the command logic. Retrieves the player's builder tool settings and delegates the exit action to the BrushConfigCommandExecutor. Complexity is constant time relative to this class, but the delegated calls may have their own complexity. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in code. It is invoked automatically by the server's command dispatcher when a player types the corresponding command into the chat console.

```
// Player-invoked command in-game
/exit
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BrushConfigExitCommand()`. Instances are managed exclusively by the command registration system. Creating an instance manually serves no purpose as it will not be registered to handle any input.
- **Manual Invocation:** Avoid calling the `execute` method directly. This bypasses the server's command processing pipeline, including permission checks, context validation, and error handling. All command execution must flow through the central command dispatcher.

## Data Pipeline
This component acts as a control-flow trigger rather than a data processor. The flow represents a command being translated into a state change.

> Flow:
> Player Chat Input (`/exit`) -> Server Command Parser -> Command Dispatcher -> **BrushConfigExitCommand.execute()** -> PrototypePlayerBuilderToolSettings lookup -> BrushConfigCommandExecutor.exitExecution() -> Player State Update<ctrl63>

