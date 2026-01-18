---
description: Architectural reference for WorldCommand
---

# WorldCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world
**Type:** Transient

## Definition
```java
// Signature
public class WorldCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The WorldCommand class is a **Composite Command Aggregator**. It does not implement any game logic itself. Instead, its sole architectural purpose is to act as a hierarchical container, grouping a suite of related world-management commands under a single, user-facing namespace: *world*.

This class follows the Composite design pattern. It extends AbstractCommandCollection, which provides the framework for registering child commands (the "leaves" of the pattern). By centralizing all world-related sub-commands—such as WorldListCommand, WorldSaveCommand, and WorldPerfCommand—it simplifies the global command registry and provides a cohesive command-line interface for server administrators.

When the server's command processor receives input like `/world save`, it first routes the request to this WorldCommand instance. WorldCommand then delegates the final execution to the appropriate registered sub-command, in this case, WorldSaveCommand.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's central CommandManager or a similar registry during the server bootstrap or plugin loading phase. The system typically discovers and registers all command classes at startup.
- **Scope:** The WorldCommand instance is a long-lived object. It persists for the entire server session, held as a reference within the command processing system.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection only when the server's command registry is cleared, which typically occurs during a full server shutdown.

## Internal State & Concurrency
- **State:** The internal state of this class is effectively **immutable** after construction. The constructor populates an internal list of sub-command objects. This list is not modified during the object's lifetime. The class itself holds no mutable game state.
- **Thread Safety:** This class is inherently thread-safe. Its state does not change after initialization. However, command execution is typically handled by a single, main server thread to prevent race conditions in game logic. Concurrency concerns are therefore delegated to the individual sub-commands it contains, not this container class.

## API Surface
The primary public contract is the constructor, which is used by the command system for initialization. All other behavior is inherited from AbstractCommandCollection and is part of the internal command framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldCommand() | Constructor | O(N) | Constructs the command group. Instantiates and registers N sub-commands. Complexity is proportional to the number of sub-commands being added. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by game systems. It is designed to be automatically discovered and registered by the server's command handling framework. A developer extending the system would add a new sub-command to its constructor.

```java
// Example of how the command system might register this class
// This code would exist within a CommandManager, NOT in typical game logic.

CommandManager commandManager = server.getCommandManager();
commandManager.register(new WorldCommand());

// A user would then interact with it via the game client console:
// > /world list
// > /world save
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not instantiate this class manually via `new WorldCommand()` in your game logic to try and execute a command. The command system is responsible for its entire lifecycle.
- **Post-Construction Modification:** Do not attempt to retrieve an instance of this class to add or remove sub-commands at runtime. The command collection is considered sealed after its constructor completes.

## Data Pipeline
WorldCommand acts as a dispatcher in the server's command processing pipeline. It receives a parsed command request and routes it to the correct sub-component for execution.

> Flow:
> Client Input (`/world perf`) -> Network Layer -> Command Parser -> **WorldCommand (Dispatcher)** -> WorldPerfCommand (Executor) -> Performance Monitoring System

