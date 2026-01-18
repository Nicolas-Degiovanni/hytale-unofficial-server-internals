---
description: Architectural reference for TintCommand
---

# TintCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class TintCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The TintCommand class is a server-side command handler that provides a user-facing entry point for the Builder Tools tinting functionality. It acts as a translator between player input (a chat command) and the underlying asynchronous task system of the BuilderToolsPlugin.

Architecturally, this class embodies the Command Pattern. It encapsulates a request as an object, but critically, it does not contain the business logic for the operation itself. Its primary responsibilities are:

1.  **Command Registration:** Declaring the command name, description, and required permission level (Creative) to the server's command system.
2.  **Argument Parsing:** Defining and parsing the required color argument from the player's input.
3.  **Context Validation:** Verifying that the player is in a valid state to perform a builder operation by checking against PrototypePlayerBuilderToolSettings.
4.  **Delegation:** Translating the parsed input into a functional task and submitting it to the BuilderToolsPlugin work queue for asynchronous execution.

This decoupling is a key design choice, ensuring that the command definition layer remains lightweight and separate from the core world-modification logic, which is managed centrally by the BuilderToolsPlugin.

### Lifecycle & Ownership
-   **Creation:** An instance of TintCommand is created during server bootstrap or, more specifically, when the BuilderToolsPlugin is loaded and initialized. The plugin is responsible for instantiating and registering all its associated commands with the server's central command registry.
-   **Scope:** The TintCommand object is a long-lived instance. It persists for the entire lifecycle of the server or until the parent BuilderToolsPlugin is unloaded. A single instance handles all invocations of the `/tint` command from all players.
-   **Destruction:** The object is eligible for garbage collection only when the server shuts down or the plugin is unloaded, at which point the command registry releases its reference.

## Internal State & Concurrency
-   **State:** The class is effectively stateless with respect to its execution. It contains a single instance field, colorArg, which is initialized in the constructor and is not modified thereafter. All state required for execution (the player, the world, the command arguments) is passed into the execute method.
-   **Thread Safety:** The TintCommand instance itself is safe to be referenced from multiple threads due to its immutable internal state. The execute method, however, is designed to be called by the server's main command processing thread. It interacts with other systems, notably the BuilderToolsPlugin, by submitting a task to a queue. This `addToQueue` method is expected to be a thread-safe operation, ensuring that world modifications are processed in a controlled, sequential manner, preventing race conditions.

## API Surface
The primary contract is the `execute` method, inherited from AbstractPlayerCommand and invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Validates player state, parses arguments, and queues a tint operation. Complexity is constant as it only delegates the work; the queued task's complexity depends on the selection size. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers. It is invoked automatically by the server's command system when a player with the appropriate permissions executes the command in chat. The core integration pattern is its delegation to the BuilderToolsPlugin.

```java
// Simplified view of the delegated task creation
// This logic resides inside the execute method.

String color = this.colorArg.get(context);
int colorI = parseColor(color); // Internal color parsing logic

// The key pattern: creating a lambda and passing it to another system.
BuilderToolsPlugin.addToQueue(
    playerComponent,
    playerRef,
    (r, s, componentAccessor) -> s.tint(r, colorI, componentAccessor)
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation and Invocation:** Do not manually create an instance with `new TintCommand()` and call `execute`. Doing so bypasses the server's command registration, permission checking, and argument parsing pipeline, leading to unpredictable behavior and likely NullPointerExceptions.
-   **Assuming Synchronous Execution:** The command queues an operation but does not execute it immediately. Do not write code that depends on the world modification being complete in the line following a command dispatch. The change is applied asynchronously by the BuilderToolsPlugin.

## Data Pipeline
The flow of data for a tint operation is unidirectional, originating from player input and resulting in an asynchronous world modification.

> Flow:
> Player Chat Packet (`/tint #FF0000`) -> Server Command Parser -> **TintCommand.execute** -> Input Validation & Color Parsing -> BuilderToolsPlugin Work Queue -> Asynchronous World Tint Operation -> (Optional) Player Feedback Message

