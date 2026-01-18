---
description: Architectural reference for PrefabPathNodesCommand
---

# PrefabPathNodesCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabPathNodesCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The PrefabPathNodesCommand class is a server-side administrative command that provides a diagnostic view into the AI pathing system. It acts as a bridge between the server's Command System and the underlying world pathing data, specifically the WorldPathData resource.

Its primary role is to allow server operators or developers to inspect the structure and properties of a specific, pre-defined NPC path (an IPrefabPath). It retrieves all waypoints associated with a given path ID and formats them into a human-readable message, detailing coordinates, rotation, and other pathing metadata. This is a crucial tool for debugging NPC behavior and verifying world generation data related to AI navigation.

This command is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central command registry. It is not intended for direct programmatic use by other game systems.

### Lifecycle & Ownership
- **Creation:** A single instance of PrefabPathNodesCommand is instantiated by the server's CommandSystem during the plugin loading and command registration phase. It is discovered via reflection or an explicit registration call within its parent plugin, PathPlugin.
- **Scope:** The object instance persists for the entire server session, held within the CommandSystem's registry. However, its execution is transient; the logic within the execute method is invoked and completed for each command usage, with no state retained between calls.
- **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down or the PathPlugin is unloaded, at which point the CommandSystem clears its command registry.

## Internal State & Concurrency
- **State:** The class instance itself is effectively stateless between executions. It holds final fields (`worldgenIdArg`, `pathArg`) which are immutable definitions for argument parsing. All state related to a specific command execution, such as the parsed arguments and the constructed response message, is confined to local variables within the `execute` method's stack frame.
- **Thread Safety:** **This class is not thread-safe.** The `execute` method accesses and reads from the `World` and its associated `EntityStore`. These world-level objects are not designed for concurrent access and must only be manipulated from the main server thread. The CommandSystem guarantees that all command executions are scheduled and run on this main thread, preventing concurrency issues.

**WARNING:** Any attempt to invoke the `execute` method from an asynchronous task or a different thread will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is defined by its role as a command. The primary entry point is the `execute` method, which is invoked by the framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(N) | Fetches and displays path data. Complexity is linear, O(N), where N is the number of waypoints in the path. This method is the core logic and is invoked by the CommandSystem. |

## Integration Patterns

### Standard Usage
This class is not designed to be used programmatically. Its standard and sole intended use is through the server console or by an in-game user with sufficient permissions.

The command requires two arguments: a `worldgenId` to identify the prefab's origin, and a `uuid` to identify the specific path instance.

```sh
# Example command executed in the server console
# Usage: /nodes <worldgenId> <path_uuid>
/nodes 12 123e4567-e89b-12d3-a456-426614174000
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PrefabPathNodesCommand()`. The command system manages the lifecycle of command objects. Manual instantiation serves no purpose and the instance will not be registered to handle commands.
- **Manual Invocation:** Do not call the `execute` method directly. Bypassing the CommandSystem framework means skipping critical steps like argument parsing, permission validation, and context setup, which will result in runtime exceptions and incorrect behavior.

## Data Pipeline
The command initiates a read-only data flow from the world's storage to the user's screen and the server log. It acts as a request-response mechanism for debugging data.

> Flow:
> User Command Input -> Server Command Parser -> **PrefabPathNodesCommand.execute()** -> EntityStore -> WorldPathData Resource -> IPrefabPath Object -> Formatted Message -> CommandContext (User Output) & Server Log

