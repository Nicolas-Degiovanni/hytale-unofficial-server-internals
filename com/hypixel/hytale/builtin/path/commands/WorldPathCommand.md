---
description: Architectural reference for WorldPathCommand
---

# WorldPathCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Transient

## Definition
```java
// Signature
public class WorldPathCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The WorldPathCommand class is a structural component within the server's Command System. It embodies the **Composite Design Pattern**, functioning as a non-executable root node that aggregates a suite of related sub-commands under a unified `/worldpath` namespace.

Its primary architectural role is that of a **dispatcher** or **router**. It contains no direct command logic itself. When the server's CommandManager processes a command beginning with `worldpath`, it delegates control to this object. WorldPathCommand then inspects the subsequent arguments to identify and forward the execution request to the appropriate registered sub-command, such as WorldPathListCommand or WorldPathSaveCommand.

This design promotes high cohesion and low coupling. New functionality related to world paths can be added by creating a new command class and registering it within the WorldPathCommand constructor, requiring no changes to the core command processing system. This makes the command hierarchy extensible and maintainable.

### Lifecycle & Ownership
- **Creation:** A single instance is created during the server's bootstrap phase. This occurs when the central CommandRegistry discovers and registers all built-in command collections.
- **Scope:** The object is stateless and its lifecycle is bound to the CommandRegistry. It persists for the entire server session.
- **Destruction:** The instance is marked for garbage collection when the server shuts down and the CommandRegistry is cleared. There are no explicit cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The object is effectively **immutable** post-construction. Its internal list of sub-commands is populated once within the constructor and is not modified during runtime. It holds no player-specific or world-specific state.
- **Thread Safety:** This class is inherently **thread-safe**. As its internal state is fixed after initialization, it can be safely accessed by multiple server threads processing commands from different players concurrently without requiring locks or synchronization. The parent AbstractCommandCollection guarantees safe concurrent read access to the sub-command list.

## API Surface
The public contract of this class is fulfilled entirely by its constructor and the behavior inherited from AbstractCommandCollection. It exposes no unique public methods for programmatic interaction. Its purpose is to be registered and managed by the command system, not to be called directly by other game systems.

## Integration Patterns

### Standard Usage
This class is not designed for direct invocation. It is exclusively used during the server initialization process to register the entire suite of `/worldpath` commands.

```java
// Conceptual example of server bootstrap
// Do NOT replicate this code; it is handled by the engine.
CommandRegistry commandRegistry = server.getService(CommandRegistry.class);

// The engine instantiates and registers the collection.
// This single action makes /worldpath list, /worldpath save, etc. available.
commandRegistry.register(new WorldPathCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid creating new instances of WorldPathCommand in game logic. The command system requires only a single, shared instance registered at startup.
- **Manual Execution:** Do not attempt to retrieve this object and invoke methods on it to run a command. All command execution must flow through the central CommandManager to ensure proper context, permissions, and argument parsing are applied.

## Data Pipeline
WorldPathCommand acts as a routing point in the command processing pipeline. It receives a parsed command request and dispatches it to a more specialized handler.

> Flow:
> Player Console Input -> Network Packet -> Server Command Parser -> CommandManager -> **WorldPathCommand** (Routing) -> Sub-Command (e.g., WorldPathSaveCommand) -> World Pathing System

