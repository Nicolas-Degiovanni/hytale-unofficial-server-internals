---
description: Architectural reference for ParkourCommand
---

# ParkourCommand

**Package:** com.hypixel.hytale.builtin.parkour
**Type:** Transient

## Definition
```java
// Signature
public class ParkourCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The ParkourCommand class is a **Command Aggregator**. It does not implement any direct game logic or command execution. Instead, its sole architectural purpose is to serve as a container and a routing entry point for a group of related sub-commands under a single, user-facing namespace: *checkpoint*.

By extending AbstractCommandCollection, it integrates into the server's core command processing system. This design pattern enforces a hierarchical command structure, which simplifies command discovery for both players and developers. When the server's Command Dispatcher receives a command beginning with *checkpoint*, it delegates processing to this collection, which in turn routes the request to the appropriate sub-command (e.g., CheckpointAddCommand, CheckpointRemoveCommand).

This class is a foundational piece of the Parkour feature's command-line interface, providing structure and organization rather than direct functionality.

### Lifecycle & Ownership
- **Creation:** An instance of ParkourCommand is created by the server's command registration service during the server bootstrap sequence or when the Parkour game module is loaded. This is typically an automated process driven by class discovery mechanisms.
- **Scope:** The object's lifecycle is bound to the server's command registry. It persists for the entire duration of the server session, or as long as the Parkour module remains active.
- **Destruction:** The instance is marked for garbage collection when the command registry is cleared during server shutdown or when the parent module is unloaded.

## Internal State & Concurrency
- **State:** The internal state of this class is **effectively immutable** after the constructor completes. Its state consists of the list of registered sub-commands, which is populated once and is not designed to be modified thereafter.
- **Thread Safety:** This class is **thread-safe**. Since its internal collection of sub-commands is not modified after initialization, it can be safely accessed by multiple command-processing threads simultaneously. The thread safety of the *executed sub-commands* is a separate concern managed by those individual classes.

## API Surface
The public contract of this class is minimal and intended for use only by the command registration system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ParkourCommand() | constructor | O(1) | Constructs the command collection. Sets the base command name to "checkpoint" and registers all constituent parkour sub-commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in game logic. It is automatically discovered and managed by the server's command system. A theoretical manual registration would look like the following, though this is handled by the engine.

```java
// This is a conceptual example of how the engine uses the class.
// Do not replicate this in game scripts.
CommandSystem commandSystem = server.getCommandSystem();
commandSystem.register(new ParkourCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of ParkourCommand within game logic. The command system is responsible for its lifecycle. Creating a rogue instance will have no effect and wastes memory.
- **Extending for Logic:** Do not extend this class to add new game logic. To add a new sub-command, create a new class that implements the appropriate command interface and add it via the `addSubCommand` method within this class's constructor.

## Data Pipeline
ParkourCommand acts as a routing step in the server's command processing pipeline. It receives a parsed command and dispatches it to the correct implementation.

> Flow:
> Player Chat Input (`/checkpoint add`) -> Server Network Listener -> Command Parser -> Command Dispatcher -> **ParkourCommand** (Router) -> CheckpointAddCommand (Executor) -> World State Change

