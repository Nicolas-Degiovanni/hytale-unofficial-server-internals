---
description: Architectural reference for RepulsionCommand
---

# RepulsionCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.component.repulsion
**Type:** Transient

## Definition
```java
// Signature
public class RepulsionCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The RepulsionCommand class is a structural component within the server's command system. It functions as a **Command Group**, a non-executable parent command that organizes and provides a common namespace for a set of related sub-commands. Its sole architectural purpose is to act as a dispatcher.

When a user executes a command beginning with *repulsion*, the server's command handler routes the request to this object. RepulsionCommand then parses the next argument (e.g., *add* or *remove*) and delegates the final execution to the corresponding registered sub-command object, such as RepulsionAddCommand or RepulsionRemoveCommand.

This pattern is critical for creating a clean and hierarchical command-line interface for server administrators and developers, preventing pollution of the global command namespace.

### Lifecycle & Ownership
- **Creation:** A single instance of RepulsionCommand is created by the server's command registration service during the server bootstrap sequence. This is typically part of an automated discovery process that scans the classpath for command definitions.
- **Scope:** The instance is registered with and owned by a central command registry. It persists for the entire lifecycle of the server session.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the command registry is cleared during server shutdown.

## Internal State & Concurrency
- **State:** This object is effectively **immutable** after its constructor completes. Its internal state consists of a list of child command objects (RepulsionAddCommand, RepulsionRemoveCommand), which is populated once and never modified. It holds no session-specific or mutable runtime data.
- **Thread Safety:** This class is **not thread-safe** and must only be accessed by the main server thread. The server's command processing engine is responsible for ensuring that all command execution is serialized, preventing race conditions related to game state modification. Any attempt to interact with this object from an asynchronous task is a severe anti-pattern.

## API Surface
The primary interaction with this class is its constructor, used by the framework during initialization. The public contract for execution is defined by its parent, AbstractCommandCollection, and is not intended for direct developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RepulsionCommand() | Constructor | O(1) | Constructs the command group, setting its name and description. Registers all child sub-commands. |

## Integration Patterns

### Standard Usage
Developers do not interact with instances of this class directly. The primary integration pattern is declarative: the class is created, and the server's command discovery mechanism automatically registers it.

The intended "user" is a server administrator or player executing the command from the game console.

**Example Console Usage:**
```console
/repulsion add <entity_id> <strength>
/repulsion remove <entity_id>
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create an instance of RepulsionCommand using *new*. The command system is responsible for its lifecycle. A manually created instance will not be registered and will have no effect.
- **Stateful Logic:** Do not add mutable fields or state to this class. A command group must be stateless, serving only to dispatch to its children.

## Data Pipeline
RepulsionCommand acts as a routing node in the data flow of command processing. It receives a partially parsed command string and directs it to the appropriate handler based on the first argument.

> Flow:
> Player Console Input -> Server Network Listener -> Command Parsing Service -> **RepulsionCommand** (Dispatch Logic) -> RepulsionAddCommand (Execution) -> Entity Component System Update

