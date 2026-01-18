---
description: Architectural reference for EditLineCommand
---

# EditLineCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Transient

## Definition
```java
// Signature
public class EditLineCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The EditLineCommand class is a server-side command handler responsible for interpreting and processing the in-game `/editline` command. It serves as a crucial translation layer, converting raw player chat input into a structured, asynchronous world modification task.

Architecturally, this class implements the **Command Pattern**. It encapsulates all information needed to perform an action—in this case, drawing a line of blocks. However, it does not execute the world modification directly. Instead, it acts as a **Dispatcher**, validating and packaging the request before submitting it to the BuilderToolsPlugin's processing queue.

This decoupling is critical for server performance. Potentially long-running world edits are offloaded from the main command processing thread, preventing server stalls and ensuring a responsive player experience. The class's primary responsibility is argument parsing and task delegation, not direct world manipulation.

## Lifecycle & Ownership
- **Creation:** A single prototype instance of EditLineCommand is created and registered by the server's command system during plugin initialization. This registration process typically uses reflection to discover all classes that extend a command base class like AbstractPlayerCommand.

- **Scope:** The registered prototype instance persists for the lifetime of the server or until the owning plugin is disabled. However, the execution context, including parsed arguments, is transient and scoped exclusively to a single command invocation. The object itself is stateless between calls.

- **Destruction:** The prototype instance is destroyed when the server shuts down or the plugin is unloaded. It is not created and destroyed on a per-command basis.

## Internal State & Concurrency
- **State:** An instance of EditLineCommand is effectively **immutable** after its construction. Its state consists of final fields representing the definitions of its command arguments (e.g., startArg, endArg, materialArg). This state defines the command's signature but does not hold data from any specific execution. All execution-specific data is passed via the CommandContext parameter in the execute method.

- **Thread Safety:** This class is inherently thread-safe. The execute method is invoked by the server's command processing thread. Since the object has no mutable instance fields, and all work is either local to the method stack or delegated to a thread-safe queue (BuilderToolsPlugin.addToQueue), there are no concurrency hazards within this class.

    **WARNING:** While this class is thread-safe, the lambda function it enqueues will be executed on a different thread managed by the BuilderToolsPlugin. That execution context has its own concurrency and safety rules that must be respected.

## API Surface
The primary contract is the `execute` method, which is invoked by the command system framework. Direct invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Parses arguments from the context and enqueues an asynchronous world edit operation. This method is non-blocking and returns immediately. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation or invocation by developers. It is automatically discovered and managed by the server's command system. The standard interaction is a player executing the command in-game.

The core integration pattern is the delegation to the BuilderToolsPlugin queue, which is the correct way to perform large-scale, deferred world edits.

```java
// The execute method demonstrates the primary integration pattern:
// packaging logic into a lambda and submitting it to a work queue.

BuilderToolsPlugin.addToQueue(
    playerComponent,
    playerRef,
    (r, s, componentAccessor) -> s.editLine(
        // ... arguments parsed from command context ...
    )
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new EditLineCommand()`. The command system handles registration and lifecycle. Manual instantiation creates an object that is not registered to handle any commands.
- **Direct Invocation:** Do not call the `execute` method directly. This bypasses critical framework functionality, including permission checks, argument parsing, and context setup, which will lead to NullPointerExceptions and incorrect behavior.
- **Synchronous World Edits:** Re-implementing this command to perform world edits directly within the `execute` method is a severe anti-pattern. It would block the server thread, causing significant lag for all players during large operations.

## Data Pipeline
The flow of data for an `/editline` command begins with player input and ends with a change in world state, with this class acting as a key intermediary.

> Flow:
> Player Chat Input -> Server Network Parser -> Command System Dispatcher -> **EditLineCommand.execute()** -> BuilderToolsPlugin Task Queue -> World Edit Executor Thread -> World Storage Update -> Network Packet to Clients (for visual update)

