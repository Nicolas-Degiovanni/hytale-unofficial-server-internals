---
description: Architectural reference for StackCommand
---

# StackCommand

**Package:** com.hypixel.hytale.builtin.buildertools.commands
**Type:** Handler

## Definition
```java
// Signature
public class StackCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The StackCommand class is a server-side command handler responsible for interpreting and executing the builder tool *stack* operation. It serves as the primary bridge between player chat input and the deferred world-modification logic managed by the BuilderToolsPlugin.

Architecturally, this class is not a worker; it is a dispatcher. Its sole responsibility is to parse arguments, validate the player's state and selection, and then enqueue a task for asynchronous execution. This decouples the immediate, low-latency command processing from the potentially long-running, resource-intensive world manipulation.

The command supports multiple argument structures (e.g., with or without a direction) through the use of private inner classes, such as StackWithCountCommand and StackWithDirectionAndCountCommand. These classes act as usage variants, allowing the core command system to match different input patterns while centralizing the final execution logic in a single static helper method, executeStack.

Interaction with the game world is performed through the Entity Component System (ECS) via the provided Store and Ref objects, which grant access to player components like HeadRotation for determining relative direction.

### Lifecycle & Ownership
- **Creation:** A single instance of StackCommand is created by the server's command registration system when the BuilderToolsPlugin is loaded. It is not created on a per-request basis.
- **Scope:** The StackCommand object is a long-lived handler. It persists for the entire lifecycle of the server or until the parent plugin is unloaded.
- **Destruction:** The object is eligible for garbage collection only upon server shutdown or when the BuilderToolsPlugin is disabled and its commands are deregistered.

## Internal State & Concurrency
- **State:** The StackCommand object itself is effectively stateless regarding command execution. It holds immutable references to argument definition objects (e.g., emptyFlag, spacingArg) which are configured once in the constructor. All state related to a specific command invocation is passed via the CommandContext.
- **Thread Safety:** This class is thread-safe under the assumption that the command system invokes the execute method from a single, controlled thread (typically the main server thread). The critical operation, world modification, is made safe by delegating it to the BuilderToolsPlugin's internal queue. This queue serializes world-editing tasks, preventing race conditions and ensuring world state consistency.

**WARNING:** Directly invoking the execute method from multiple threads is not supported and will lead to unpredictable behavior and potential world corruption.

## API Surface
The public API is defined by its role as a command handler within the server framework. Direct invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Framework entry point. Parses arguments from the context and enqueues a stack operation. Throws exceptions on invalid arguments. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly via code. It is invoked by the server's command system when a player with appropriate permissions (GameMode.Creative) types the command into chat.

*Example Player Input:*
```
/stack up 10 --empty -spacing:1
```

The framework then routes this to the StackCommand handler.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new StackCommand()` in your own logic. The command system manages the lifecycle of this object. Manually creating an instance bypasses registration and will have no effect.
- **Synchronous Assumption:** Do not write code that assumes the stack operation is complete after the command executes. The world modification is deferred and asynchronous. There is no guarantee of when the operation will finish.
- **Directly Calling executeStack:** The private static `executeStack` method is an internal implementation detail. Bypassing the main `execute` method will skip critical context-based parsing and validation.

## Data Pipeline
The flow of data for a stack command begins with player input and ends with a change in the world state. The StackCommand acts as an early component in this pipeline, responsible for validation and delegation.

> Flow:
> Player Chat Input -> Network Layer -> Server Command Parser -> **StackCommand.execute** -> Validation & Argument Parsing -> BuilderToolsPlugin.addToQueue(task) -> World Edit Thread -> World Storage Update

