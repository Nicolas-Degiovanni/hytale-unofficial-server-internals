---
description: Architectural reference for ChunkTrackerCommand
---

# ChunkTrackerCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient

## Definition
```java
// Signature
public class ChunkTrackerCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The ChunkTrackerCommand is a diagnostic command within the server's Command System framework. Its primary function is to provide a real-time snapshot of the chunk loading subsystem's state, both for a specific player and for the world at large.

Architecturally, this class serves as a read-only interface between an administrator's input and the complex, stateful systems of the world simulation. By extending AbstractTargetPlayerCommand, it delegates the complex logic of parsing player targets, handling permissions, and setting up execution context to the base framework. This allows the ChunkTrackerCommand to focus exclusively on its core responsibility: querying the Entity Component System (ECS) for the ChunkTracker component and the World for its ChunkStore, and then presenting this data to the user.

It is a terminal node in the command processing chain, translating low-level engine metrics into a human-readable, localized message.

## Lifecycle & Ownership
- **Creation:** A single instance of ChunkTrackerCommand is instantiated by the server's CommandRegistry during the server bootstrap phase. It is discovered via reflection and registered as the handler for the "tracker" subcommand.
- **Scope:** The command object itself is a long-lived singleton that persists for the entire server session. However, each execution of the command is a transient, short-lived operation.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. It contains no mutable instance fields. The only field, MESSAGE_COMMANDS_CHUNK_TRACKER_SUMMARY, is a static final constant, making it immutable and shared across all potential executions. All data required for an operation is provided via method arguments to the execute method.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. However, the execution context is not. The execute method reads from core game state objects (Store, World) which are not thread-safe. The command system framework **MUST** ensure that this method is always invoked on the main server tick thread to prevent race conditions and data corruption. Direct invocation from asynchronous tasks is strictly forbidden.

## API Surface
The public contract is defined by the inherited `execute` method. Direct interaction is discouraged; use the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, sourceRef, ref, playerRef, world, store) | void | O(1) | Queries ECS and World state for chunk statistics. Sends a formatted summary message to the command source. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly in code. It is invoked by the server's command dispatcher in response to a user typing the command in-game or in the server console.

Example user interaction:
```
/chunk tracker Player123
```
This input triggers the command system, which resolves the command to this class, finds the entity for "Player123", and invokes the `execute` method with the correct context.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ChunkTrackerCommand()`. The command system manages the lifecycle of command handlers. Manually creating an instance will result in an object that is not registered with the dispatcher and is therefore useless.
- **Manual Invocation:** Do not call the `execute` method directly. Bypassing the command dispatcher means you circumvent critical infrastructure, including permission checks, argument validation, and thread synchronization. This can lead to security vulnerabilities and server instability.

## Data Pipeline
The flow of data for a typical command execution is unidirectional, from user input to user feedback.

> Flow:
> User Input (`/chunk tracker`) -> Network Layer -> Command Dispatcher -> **ChunkTrackerCommand.execute()** -> ECS Store & World Services -> Data Aggregation -> Message Formatter -> CommandContext -> Network Layer -> Client Chat UI

