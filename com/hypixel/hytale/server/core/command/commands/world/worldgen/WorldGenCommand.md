---
description: Architectural reference for WorldGenCommand
---

# WorldGenCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.worldgen
**Type:** Transient

## Definition
```java
// Signature
public class WorldGenCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The WorldGenCommand class is not an executable command itself, but rather a **Command Collection**. Its primary architectural function is to act as a grouping mechanism or namespace for a set of related sub-commands concerning the server's world generation system.

By extending AbstractCommandCollection, it inherits the logic required to parse command-line arguments and delegate execution to one of its registered children, such as WorldGenBenchmarkCommand or WorldGenReloadCommand. This design pattern promotes a clean and hierarchical command structure, improving discoverability for server administrators and modularity for developers. It serves as a structural routing component within the server's central Command Dispatch System.

## Lifecycle & Ownership
- **Creation:** An instance of WorldGenCommand is created during the server's bootstrap sequence. It is instantiated by a central CommandManager or a similar registry service responsible for discovering and loading all available server commands.
- **Scope:** The object instance persists for the entire duration of the server session. Its lifecycle is directly bound to the CommandManager that holds a reference to it.
- **Destruction:** The object is marked for garbage collection when the CommandManager is cleared, which typically occurs during a full server shutdown or a comprehensive plugin reload cycle. It manages no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state of this class is configured exclusively within its constructor and is **effectively immutable** post-instantiation. This state consists of its primary name ("worldgen"), its alias ("wg"), and a list of its sub-command children. The class itself is stateless regarding command execution; it only holds structural configuration.
- **Thread Safety:** This class is inherently **thread-safe**. As its internal state is immutable after construction, it can be safely accessed by the Command Dispatch System from any thread without synchronization.

    **Warning:** While the WorldGenCommand router is thread-safe, the sub-commands it contains may have their own distinct concurrency models and state management. Thread safety guarantees do not automatically extend to its children.

## API Surface
The primary API for this class is not programmatic but rather the command-line interface it exposes to the server console or in-game user. It defines the `worldgen` command and its sub-commands.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| worldgen | Command Group | O(log N) | Acts as a namespace for world generation commands. Delegates to a sub-command. |
| worldgen benchmark | Sub-Command | - | Executes the WorldGenBenchmarkCommand. |
| worldgen reload | Sub-Command | - | Executes the WorldGenReloadCommand. |
| wg | Alias | O(log N) | An alias for the `worldgen` command group. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation. It is designed to be discovered and registered with the server's central command handling system during initialization.

```java
// Example of registration within a CommandManager
// This code would exist in a higher-level bootstrap service.

CommandManager commandManager = serverContext.getCommandManager();
commandManager.register(new WorldGenCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not instantiate this class for any purpose other than registering it with the CommandManager. A standalone instance has no functionality.
- **Manual Execution:** Never call the `execute` method (inherited from AbstractCommandCollection) directly. This method requires a specific execution context (e.g., CommandSender, arguments) that is only supplied by the server's Command Dispatch System. Bypassing the dispatcher will lead to NullPointerExceptions and undefined behavior.

## Data Pipeline
The WorldGenCommand class acts as a routing step in the server's command processing pipeline. It receives parsed input and dispatches it to the appropriate handler based on the first argument.

> Flow:
> User Input (`/wg benchmark`) -> Server Network Layer -> Command Parser -> Command Dispatcher -> **WorldGenCommand** (Router) -> WorldGenBenchmarkCommand (Executor) -> World Generation Service -> Result Feedback

