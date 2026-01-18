---
description: Architectural reference for InteractionCommand
---

# InteractionCommand

**Package:** com.hypixel.hytale.server.core.modules.interaction.commands
**Type:** Transient

## Definition
```java
// Signature
public class InteractionCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The InteractionCommand class is a concrete implementation of the Composite Command pattern. It functions as a non-terminal node within the server's command processing tree, acting as a namespace and dispatcher for a group of related sub-commands.

Its sole architectural purpose is to aggregate all commands related to the server's *Interaction System* under a single, user-facing entry point: **interaction**. It does not contain any execution logic itself. When the server's command processor identifies the **interaction** keyword, it delegates control to this object. InteractionCommand then parses the subsequent arguments to identify and invoke the appropriate sub-command, such as InteractionRunCommand or InteractionClearCommand.

This design decouples the high-level command router from the specific implementations of interaction-related tasks, promoting modularity and extensibility of the command system.

### Lifecycle & Ownership
- **Creation:** Instantiated once during the server's bootstrap sequence by a central command registration service. This object is not created on-demand per command execution.
- **Scope:** Application-scoped. It persists in memory for the entire duration of the server session, held as a reference within the global command registry.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only upon server shutdown or when its parent module is unloaded, at which point the central command registry is cleared.

## Internal State & Concurrency
- **State:** Effectively **Immutable** post-construction. The collection of sub-commands is populated exclusively within the constructor and is not designed to be modified at runtime. The class holds no mutable, session-specific data.

- **Thread Safety:** This class is inherently **Thread-Safe**. Its immutable-after-construction nature ensures that multiple threads, such as those processing commands from different players concurrently, can safely traverse its structure without race conditions or the need for explicit locking. All state is structural and read-only after initialization.

    **Warning:** While this container class is thread-safe, the sub-commands it dispatches to (e.g., InteractionRunCommand) may have their own distinct state and concurrency constraints that must be respected.

## API Surface
The public contract of InteractionCommand is almost entirely inherited from its parent, AbstractCommandCollection. It defines no new public methods. Its primary role is declarative, established through its constructor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| InteractionCommand() | Constructor | O(1) | Registers the command name, alias, and aggregates all child sub-commands. This is the sole point of configuration. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by application code. It is designed to be discovered and registered by the server's central command system at startup. The correct integration is purely declarative.

```java
// Example of how the command system might register this class
// This code would exist in a central command registry or module loader.

CommandRegistry registry = server.getCommandRegistry();
registry.register(new InteractionCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of InteractionCommand to execute a command. The server's command processor is the sole intended user. Creating an instance manually will result in a disconnected object that is not part of the command tree.

- **State Modification:** Do not attempt to use reflection or other means to modify the internal list of sub-commands after the object has been constructed. The system relies on this list being stable for the server's lifetime.

## Data Pipeline
InteractionCommand acts as a routing step in the flow of data from player input to system execution. It validates the primary command token and delegates the rest of the payload to the appropriate handler.

> Flow:
> Player Chat Input (`/interaction run ...`) -> Network Packet -> Server Command Parser -> **InteractionCommand** (Matches "interaction") -> Sub-Command Dispatcher -> InteractionRunCommand (Receives "run ...") -> Interaction System Logic

