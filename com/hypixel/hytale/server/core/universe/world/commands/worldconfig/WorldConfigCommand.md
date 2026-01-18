---
description: Architectural reference for WorldConfigCommand
---

# WorldConfigCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.worldconfig
**Type:** Configuration Object

## Definition
```java
// Signature
public class WorldConfigCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The WorldConfigCommand class is a **Command Collection**, a structural component within the server's command system. It does not implement any direct game logic itself. Instead, it acts as a container or namespace for a group of related sub-commands that manage world-specific configurations.

Architecturally, this class follows the **Composite Design Pattern**. It forms a non-terminal node in the server's command tree. The primary command name, *config*, serves as a parent, and its children are the concrete command implementations like WorldConfigSeedCommand and WorldConfigSetPvpCommand. This pattern is crucial for creating a clean, hierarchical, and extensible command-line interface for server administrators, preventing pollution of the global command namespace.

Its behavior is almost entirely inherited from the AbstractCommandCollection base class, which provides the core logic for parsing sub-command arguments and dispatching execution to the appropriate child command.

### Lifecycle & Ownership
- **Creation:** An instance of WorldConfigCommand is created once during the server's bootstrap sequence. A central command registry or manager is responsible for discovering and instantiating all command classes, including this one.
- **Scope:** The object is a long-lived singleton in practice, though not enforced by the class itself. It persists in the server's command registry for the entire server session.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down and the command registry is cleared.

## Internal State & Concurrency
- **State:** The class is effectively **immutable** after construction. Its internal state consists of a list of its registered sub-commands. This list is populated exclusively within the constructor and cannot be modified thereafter. It holds no mutable world state or session data.
- **Thread Safety:** This class is inherently **thread-safe**. Because its internal state is fixed upon creation, it can be safely accessed by any thread. However, the execution of the sub-commands it contains is a separate concern. The command system guarantees that the execution logic of child commands is dispatched onto the main server thread to prevent race conditions when modifying world state.

## API Surface
The primary public contract of this class is its constructor, which defines its identity and composition. All other interactive behavior is handled through the command system via its parent, AbstractCommandCollection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldConfigCommand() | Constructor | O(1) | Instantiates the command collection, setting its primary name to *config* and registering all its child commands. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly at runtime. It is designed to be automatically discovered and registered by the command system. The primary interaction is by a server administrator executing the command from the console or in-game.

The following example illustrates how a command registry might instantiate and register this collection during server initialization.

```java
// Hypothetical registration process during server startup
CommandRegistry commandRegistry = server.getService(CommandRegistry.class);
AbstractCommand worldRootCommand = commandRegistry.getRoot("world");

// The WorldConfigCommand is added as a child of the 'world' command
worldRootCommand.addSubCommand(new WorldConfigCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Execution:** Never instantiate this class with the intent of directly invoking a command. The command system is responsible for parsing input and dispatching to the correct handler. Creating an instance manually bypasses this entire system.
- **Runtime Modification:** Do not attempt to retrieve this object from the registry to add or remove sub-commands at runtime. The class is not designed for dynamic composition and such actions would lead to unpredictable behavior. New commands should be added declaratively as new classes.

## Data Pipeline
The WorldConfigCommand acts as a dispatcher in the data flow of command execution. It receives a partially parsed command and routes it to the correct sub-handler.

> Flow:
> Server Console Input (`/world config pvp false`) -> Network Layer -> Command Parser -> Command Registry (Finds `world` command) -> `world` command (Delegates to `config` sub-command) -> **WorldConfigCommand** (Delegates to `pvp` sub-command) -> WorldConfigSetPvpCommand (Executes logic) -> World State Modification

