---
description: Architectural reference for PrefabSpawnerCommand
---

# PrefabSpawnerCommand

**Package:** com.hypixel.hytale.server.core.modules.prefabspawner.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabSpawnerCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PrefabSpawnerCommand class serves as a command group, or namespace, for all server commands related to the Prefab Spawner module. It does not implement any command logic itself; instead, it acts as a structural container that aggregates and routes to more specific sub-commands like `get`, `set`, and `weight`.

By extending AbstractCommandCollection, it integrates into the server's command system, which is responsible for parsing user input and dispatching it to the appropriate handler. This design pattern simplifies the command hierarchy by grouping related functionality under a single, memorable name (`prefabspawner`) and its alias (`pspawner`). This class is a foundational piece of the server's administrative and debugging toolset, providing a clear entry point for manipulating the prefab spawning system.

### Lifecycle & Ownership
- **Creation:** A single instance is created by the server's central CommandRegistry during the server bootstrap or module loading phase. The system scans for and instantiates all command definitions at startup.
- **Scope:** The object instance persists for the entire lifecycle of the server session. It is part of the server's static command definition tree.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server shuts down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively **immutable** post-construction. Its internal state, which consists of its name, alias, and a list of registered sub-commands, is fully configured within the constructor and is not modified during runtime. It is a stateless routing component.
- **Thread Safety:** The class is inherently **thread-safe**. Its immutable nature allows it to be safely accessed by multiple command processing threads without synchronization. Responsibility for thread safety during command execution is delegated to the individual sub-command classes.

## API Surface
The public contract of this class is fulfilled almost entirely by its constructor, which serves as its configuration entry point.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PrefabSpawnerCommand() | Constructor | O(1) | Initializes the command group, setting its primary name, description key, alias, and registering all child sub-commands. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly at runtime. It is designed to be registered with the server's command system during initialization.

```java
// Example of registration during server startup
CommandRegistry commandRegistry = serverContext.getCommandRegistry();
commandRegistry.register(new PrefabSpawnerCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never instantiate this class and attempt to call methods on it to execute a command. The server's Command Dispatcher is the sole authority for parsing and routing command input.
- **Post-Construction Modification:** Do not attempt to add sub-commands or aliases after the object has been constructed. The command tree is considered static after the server's bootstrap phase. Modifying it at runtime can lead to unpredictable behavior and race conditions.

## Data Pipeline
This class acts as a routing node in the server's command processing pipeline. It receives control from the dispatcher and passes it to a more specific sub-command handler.

> Flow:
> Player Chat Input -> Server Network Layer -> Command Dispatcher -> **PrefabSpawnerCommand** (Routing) -> PrefabSpawnerSetCommand (Execution) -> Prefab Spawner Module -> World State Update

