---
description: Architectural reference for EntitySnapshotLengthCommand
---

# EntitySnapshotLengthCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.entity.snapshot
**Type:** Transient

## Definition
```java
// Signature
public class EntitySnapshotLengthCommand extends CommandBase {
```

## Architecture & Concepts
The EntitySnapshotLengthCommand serves as a high-level, user-facing interface for configuring a low-level server performance parameter. It is a leaf node within the server's Command System, designed to be invoked by administrators to dynamically alter the duration of the entity snapshot history.

Architecturally, this class is a classic example of a *Configuration Command*. Its sole purpose is to receive a validated input from a trusted source and apply it to a static configuration variable within a core engine module, in this case, the SnapshotSystems. This pattern decouples the user input and parsing logic from the core system being configured, allowing the SnapshotSystems to remain unaware of the command framework.

**WARNING:** This command directly mutates global static state (`SnapshotSystems.HISTORY_LENGTH_NS`). This is a powerful but potentially hazardous operation that affects the memory footprint and debugging capabilities of the entire server.

### Lifecycle & Ownership
- **Creation:** A single instance of EntitySnapshotLengthCommand is instantiated by the CommandRegistry during server bootstrap. The registry scans for classes extending CommandBase and registers them for runtime invocation.
- **Scope:** The command object is a long-lived singleton, persisting for the entire server session, managed by the CommandRegistry. The execution context, however, is ephemeral; a new CommandContext is created for each command invocation.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only upon server shutdown when the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only instance field, lengthArg, is an immutable definition object used for argument parsing. The state it modifies is external, belonging to the SnapshotSystems class.
- **Thread Safety:** The command is not thread-safe by design. The `executeSync` method name is a strict contract indicating that it **must** be invoked on the main server thread. The Command System guarantees this execution context. Any attempt to invoke this method from an asynchronous task will lead to severe race conditions when modifying the non-atomic `SnapshotSystems.HISTORY_LENGTH_NS` static field.

## API Surface
The primary API is not programmatic but rather its registration within the command system. Developers do not interact with this class's methods directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(1) | Executes the command logic. Reads the length argument, converts it to nanoseconds, and sets the global snapshot history length. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is invoked by the server's command handler in response to user input from the console or an in-game chat.

A server administrator would use this command as follows:
```
/entity snapshot length 10000
```
This command sets the entity snapshot history to 10,000 milliseconds (10 seconds). The framework handles parsing, routing, and invoking the `executeSync` method on the correct instance.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new EntitySnapshotLengthCommand()`. It will not be registered with the command system and will be a useless, inert object.
- **Asynchronous Invocation:** Calling `executeSync` from a worker thread or any context other than the main server thread is a critical error. This bypasses the framework's synchronization guarantee and will corrupt the state of the SnapshotSystems, likely leading to server instability or crashes.

## Data Pipeline
The flow of data for this command is unidirectional, from user input to system state modification, with a simple feedback message.

> Flow:
> User Input (`/entity snapshot length 5000`) -> Network Layer -> Command Parser -> **EntitySnapshotLengthCommand**.executeSync() -> `SnapshotSystems.HISTORY_LENGTH_NS` (Static Field Write)
>
> Feedback Flow:
> **EntitySnapshotLengthCommand** -> `context.sendMessage()` -> Message Formatter -> Network Layer -> Client Console Output

