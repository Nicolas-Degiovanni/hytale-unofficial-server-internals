---
description: Architectural reference for PlayerCameraSubCommand
---

# PlayerCameraSubCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.camera
**Type:** Transient Component

## Definition
```java
// Signature
public class PlayerCameraSubCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The PlayerCameraSubCommand class is a structural component within the server's command system. It does not implement any direct game logic itself. Instead, it functions as a **Composite Command** or a command namespace, grouping related camera-manipulation commands under the single entry point: `/camera`.

By extending AbstractCommandCollection, it inherits the responsibility of managing a list of child commands. When the server's command dispatcher routes a command beginning with "camera", this class is responsible for parsing the next argument (e.g., "reset", "topdown") and delegating the execution to the appropriate registered sub-command.

This pattern is critical for creating a clean, hierarchical, and discoverable command structure for server administrators and players, preventing the pollution of the global command namespace.

### Lifecycle & Ownership
- **Creation:** Instantiated once during server bootstrap by the Command Registry or a similar service responsible for discovering and registering all available server commands.
- **Scope:** Singleton-like in practice, but not by pattern. A single instance persists for the entire server session once registered. It is not created on-demand per command execution.
- **Destruction:** The instance is dereferenced and eligible for garbage collection when the Command Registry is cleared, typically during server shutdown.

## Internal State & Concurrency
- **State:** This class is effectively **immutable** after construction. Its internal state consists of a list of sub-command objects. This list is populated exclusively within the constructor and is not modified during the server's operational lifecycle.
- **Thread Safety:** The class is inherently **thread-safe**. Its immutable nature ensures that multiple threads (e.g., from a command execution worker pool) can safely access it simultaneously to look up sub-commands without risking race conditions or data corruption.

## API Surface
The primary public contract is fulfilled by its constructor, which serves as the configuration point for the command collection. All execution logic is handled through interfaces implemented by its parent, AbstractCommandCollection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayerCameraSubCommand() | Constructor | O(N) | Initializes the command collection, setting its name ("camera") and registering N sub-commands. |

## Integration Patterns

### Standard Usage
Developers do not interact with an instance of this class directly. The primary pattern is to create a similar class to group a new set of commands and register it with the server's central command system.

```java
// Example of registering a new command collection during server initialization
CommandRegistry registry = server.getCommandRegistry();

// The system instantiates and registers the collection
registry.register(new PlayerCameraSubCommand());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate this class for direct use with `new PlayerCameraSubCommand()`. The command system is responsible for its lifecycle. Creating rogue instances will have no effect as they will not be registered to receive commands.
- **Post-Construction Modification:** Do not attempt to add or remove sub-commands from this collection after it has been constructed. The internal collection is not designed for runtime modification and doing so via reflection or other means can lead to unpredictable behavior.

## Data Pipeline
This class acts as a router or dispatcher within the server's command processing pipeline. It does not transform data but directs the flow of execution.

> Flow:
> Player Chat Input -> Server Network Listener -> Command Parser -> Command Registry Lookup -> **PlayerCameraSubCommand** (Dispatch) -> Sub-Command (e.g., PlayerCameraTopdownCommand) -> Game Logic Execution

