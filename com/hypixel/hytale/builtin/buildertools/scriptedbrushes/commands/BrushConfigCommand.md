---
description: Architectural reference for BrushConfigCommand
---

# BrushConfigCommand

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.commands
**Type:** Utility

## Definition
```java
// Signature
public class BrushConfigCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The BrushConfigCommand class serves as a command router and namespace for all functionalities related to the Scripted Brush system. It does not implement any direct game logic itself. Instead, it acts as a container, aggregating multiple, more specific subcommands under a single entry point, `/scriptedbrushes`.

By extending AbstractCommandCollection, it inherits the responsibility of registering and delegating command execution. When a player executes a command like `/sb list`, the server's central command system first identifies BrushConfigCommand as the handler for the `sb` alias. BrushConfigCommand then parses the next argument, `list`, and dispatches the execution context to the corresponding registered subcommand, in this case, BrushConfigListCommand.

This pattern simplifies the global command namespace, promotes code organization by grouping related functionalities, and allows for shared properties like permission requirements to be defined at the collection level.

### Lifecycle & Ownership
- **Creation:** Instantiated once during server bootstrap. A central command registry is responsible for discovering and creating an instance of this class, typically via reflection or a service loader.
- **Scope:** Singleton-like in practice. The single instance persists for the entire server session. It is registered and held by the server's command management system.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only during server shutdown when the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless and effectively immutable after its constructor completes. Its internal state consists of a list of registered subcommand objects, which is not modified after initialization.
- **Thread Safety:** The BrushConfigCommand object itself is thread-safe due to its immutability. However, the execution of its subcommands is strictly managed by the server's command system.

    **WARNING:** All subcommands are executed on the main server thread to ensure safe, synchronous access to world state (e.g., EntityStore, World). Do not attempt to invoke command execution from asynchronous tasks without proper synchronization with the main game loop.

## API Surface
The primary interaction with this class is through its constructor during the server's command registration phase. It has no other meaningful public API for direct invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BrushConfigCommand() | Constructor | O(N) | Initializes the command collection, setting its name, aliases, and permission node. Registers N subcommands. |

The functional API is exposed to players via the in-game console as subcommands:
- **clear:** Clears the current brush configuration.
- **list:** Lists available brush configurations.
- **debugstep:** Steps through brush execution for debugging.
- **exit:** Exits the current brush configuration context.
- **load:** Loads a brush configuration from a file.
- **info:** Provides information on the current brush configuration.

## Integration Patterns

### Standard Usage
A developer does not interact with an instance of this class directly. Instead, the class is registered with the server's command handler system.

```java
// Example of how the command system might register this collection
CommandRegistry registry = server.getCommandRegistry();
registry.register(new BrushConfigCommand());

// Users then interact with it in-game:
// /sb info
// /scriptedbrushes load my_cool_brush
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Invocation:** Do not create an instance of this class to try and execute logic. The class is tightly coupled to the CommandContext provided by the command system.
    ```java
    // ANTI-PATTERN: This will not work and serves no purpose.
    BrushConfigCommand myCommand = new BrushConfigCommand();
    // myCommand.execute(...) // This method is meant for the system, not users.
    ```
- **Stateful Subcommands:** Subcommands registered with this collection should be stateless. Any state should be stored in dedicated player-specific components or world systems, as demonstrated by the `info` subcommand's reliance on `ToolOperation.getOrCreatePrototypeSettings`.

## Data Pipeline
The following illustrates the data flow when a player executes the `info` subcommand. This demonstrates the class's role as a delegator.

> Flow:
> Player Input (`/sb info`) -> Server Network Layer -> Command Parser -> **BrushConfigCommand** (Routes to `info`) -> `info` Subcommand Execution -> `ToolOperation` static lookup -> Player-specific `BrushConfig` state -> Raw String returned -> `Message` object created -> `CommandContext` sends reply -> Server Network Layer -> Player Chat UI

