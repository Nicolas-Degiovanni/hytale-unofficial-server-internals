---
description: Architectural reference for HitboxCommand
---

# HitboxCommand

**Package:** com.hypixel.hytale.server.core.modules.collision.commands
**Type:** Component

## Definition
```java
// Signature
public class HitboxCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The HitboxCommand class is a server-side command handler responsible for exposing diagnostic information about block collision data. It functions as a bridge between a privileged user (a server administrator or developer) and the low-level Asset Management system, specifically the maps containing block type and bounding box definitions.

Architecturally, it is not a core game system but rather a utility component that plugs into the server's central Command System. It follows the **Command** design pattern, encapsulating a request as an object. By extending AbstractCommandCollection, it acts as a grouping mechanism for related sub-commands, namely *extents* and *get*, which are implemented as private inner classes.

Its primary role is for debugging and data validation, allowing developers to query the calculated physical properties of blocks directly from a running server instance without needing external tools.

## Lifecycle & Ownership
- **Creation:** An instance of HitboxCommand is created and registered with the server's central command dispatcher during the server bootstrap sequence or when its parent module is loaded. It is not intended for on-demand instantiation.
- **Scope:** Session-scoped. The registered instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** The HitboxCommand component is **effectively stateless**. It does not maintain any mutable state across command executions. All data required for an operation is either passed in via the CommandContext or retrieved on-demand from global, read-only static asset maps like BlockTypeAssetMap and BlockBoundingBoxes.

- **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads concurrently. The command system guarantees that all command execution, via the executeSync method, occurs on the main server thread. This serialized access model prevents race conditions when querying the underlying asset maps, which are also designed for single-threaded access post-initialization.

    **Warning:** Bypassing the command system and invoking executeSync from a worker thread will lead to concurrency violations and server instability.

## API Surface
The primary API is not programmatic but rather the command strings registered with the server. The class exposes a collection of commands under the base name *hitbox*.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hitbox extents [threshold] | Command | O(N) | Calculates the total number of "filler" blocks needed based on hitbox dimensions for all registered block types. N is the number of block types. |
| hitbox get <hitbox> | Command | O(1) | Retrieves and displays the bounding box and detailed sub-boxes for a specific, named hitbox asset. |

## Integration Patterns

### Standard Usage
This class is not designed to be used directly as a service. Its integration into the engine is handled by the command system. A module would register an instance during its initialization phase.

```java
// Example of how the command system would register this component
// This code would exist within a central CommandRegistry or similar service.

CommandRegistry registry = server.getCommandRegistry();
registry.register(new HitboxCommand());
```

Once registered, it is invoked by users through the server console or in-game chat.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new HitboxCommand()` in general application code. The object is useless unless it is registered with the server's command dispatcher.

- **Direct Invocation:** Never call the `executeSync` method directly. This bypasses critical infrastructure provided by the command system, including argument parsing, permission checks, and context setup.

- **Stateful Modification:** Do not attempt to modify this class to store state. The command system may reuse instances, and any stored state would create unpredictable behavior and memory leaks.

## Data Pipeline
The flow of data for a typical `hitbox get` command demonstrates the component's role as a read-only interface to the asset system.

> Flow:
> User Input (`/hitbox get stone`) -> Server Command Parser -> Command Dispatcher -> **HitboxCommand** -> `HitboxGetCommand.executeSync` -> Static `BlockBoundingBoxes.getAssetMap()` -> `Message` Formatting -> `CommandContext.sendMessage` -> Network Layer -> Client UI

