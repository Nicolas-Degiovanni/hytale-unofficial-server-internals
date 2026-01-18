---
description: Architectural reference for InteractionSnapshotSourceCommand
---

# InteractionSnapshotSourceCommand

**Package:** com.hypixel.hytale.server.core.modules.interaction.commands
**Type:** Transient

## Definition
```java
// Signature
public class InteractionSnapshotSourceCommand extends CommandBase {
```

## Architecture & Concepts
The InteractionSnapshotSourceCommand is a component of the server's Command System. It provides a user-facing interface for querying the server's current configuration for the interaction snapshot source. Architecturally, this class acts as a terminal endpoint in the command processing chain, translating a user-entered command into a read operation against a global configuration state.

This class follows the Composite Pattern common in command systems by acting as a root for related sub-commands (e.g., InteractionSetSnapshotSourceCommand). When invoked without a sub-command, its default behavior is to report the current state, functioning as a "getter" for the static configuration value held in the SelectInteraction class.

## Lifecycle & Ownership
- **Creation:** A single instance is instantiated by the server's central command registry during the bootstrap phase. The framework is responsible for discovering and registering all CommandBase implementations.
- **Scope:** The instance exists for the entire duration of a server session. It is effectively a singleton managed by the command system.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection when the server shuts down and the command registry is torn down.

## Internal State & Concurrency
- **State:** This class is stateless. It holds no instance fields that change during its lifetime. All contextual data required for execution, such as the command sender, is passed via the CommandContext parameter. The state it reads is external and static (SelectInteraction.SNAPSHOT_SOURCE).
- **Thread Safety:** The method name executeSync is a strong convention indicating that this method must be called from the main server thread. It is not safe for concurrent execution. The command's safety relies on the thread safety of the external static state it accesses. **Warning:** Any system that modifies SelectInteraction.SNAPSHOT_SOURCE must use proper synchronization to avoid data races.

## API Surface
The public contract of this class is defined by its behavior within the command system, not its Java methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| snapshotsource | Command | O(1) | The primary command string. When executed, it reads and reports the current snapshot source configuration. |
| executeSync(context) | void | O(1) | The framework entry point for command execution. Directly calling this method is an anti-pattern. |

## Integration Patterns

### Standard Usage
This command is designed to be invoked by a server operator or an authorized user through a command-line interface, not programmatically.

```text
// Example of a server administrator executing the command
/snapshotsource
```
The command system processes this input, routes it to the executeSync method, and returns a translated message to the administrator's console or chat window.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new InteractionSnapshotSourceCommand()`. A command object is useless unless registered with the server's command dispatcher.
- **Stateful Logic:** Do not add mutable instance fields to this class. Commands are expected to be stateless and fully re-entrant.

## Data Pipeline
The data flow for this command is a simple user-initiated read operation.

> Flow:
> User Input (`/snapshotsource`) -> Server Command Parser -> Command Dispatcher -> **InteractionSnapshotSourceCommand.executeSync()** -> Static read of `SelectInteraction.SNAPSHOT_SOURCE` -> Message Factory -> Network Layer -> Client Chat UI

---

