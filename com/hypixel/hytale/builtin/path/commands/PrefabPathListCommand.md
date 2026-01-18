---
description: Architectural reference for PrefabPathListCommand
---

# PrefabPathListCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabPathListCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The PrefabPathListCommand is a concrete implementation of the Command Pattern, designed to integrate with the server's core command processing system. As a subclass of AbstractWorldCommand, its execution is intrinsically linked to a specific World instance, granting it contextual access to world-specific data stores.

Its primary function is to serve as a read-only diagnostic tool. It provides a user-facing interface, accessible via in-game chat or the server console, for querying the state of the NPC pathing system. Specifically, it retrieves and displays a list of all predefined, world-generated paths, known as *Prefab Paths*. This class acts as the terminal endpoint in a user query, bridging the high-level command system with the low-level WorldPathData resource, which is the authoritative source for path information within a world.

## Lifecycle & Ownership
- **Creation:** A single prototype instance of PrefabPathListCommand is created and registered with the server's command handler, typically during the bootstrap phase of the PathPlugin. This instance is managed by the command system's central registry.
- **Scope:** The prototype instance is a long-lived object, persisting for the entire server session. However, the context in which it operates—the CommandContext, World, and Store objects passed to the execute method—is ephemeral and valid only for the duration of a single command invocation.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the PathPlugin is unloaded or the server shuts down, at which point the command registry is cleared.

## Internal State & Concurrency
- **State:** PrefabPathListCommand is a stateless class. It contains no mutable instance fields and all data required for its operation is provided as arguments to the execute method. Its behavior is consistent and repeatable across multiple invocations, depending only on the state of the WorldPathData resource at the time of execution.

- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, execution is not guaranteed to be safe if invoked from an arbitrary thread. The server's command system ensures that world commands are executed on the corresponding world's main tick thread. This is a critical design constraint to prevent race conditions when accessing world-state resources like the EntityStore and WorldPathData. Direct invocation from other threads will lead to undefined behavior and potential data corruption.

## API Surface
The public contract is defined by its role as a command, primarily through the overridden execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(N) | Retrieves all prefab paths from the world's data store, where N is the number of paths. Formats this data into a human-readable message and sends it to the command issuer. Also logs the same information to the server console. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic invocation. Its public interface is the chat command it registers. Developers interact with it by registering it within a plugin's command structure.

```java
// Example of registering the command within a plugin
// This is the ONLY correct way to integrate this class.
CommandSystem commandSystem = server.getCommandSystem();
CommandNode rootNode = commandSystem.getNode("npcpath");

// The system handles instantiation and lifecycle
rootNode.addChild(new PrefabPathListCommand());
```

Once registered, a user or administrator invokes it via the server console or in-game chat:
> /npcpath list

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Invocation:** Never manually create an instance of this class and call its execute method. The command system is responsible for providing the required World and Store context. Bypassing the system will result in NullPointerExceptions or access to stale data.
    ```java
    // ANTI-PATTERN: Bypasses the command system's context injection
    PrefabPathListCommand cmd = new PrefabPathListCommand();
    cmd.execute(null, null, null); // This will fail
    ```
- **Stateful Modification:** Do not extend this class to add state. The command system reuses the same registered instance for all executions, and any stored state would create unpredictable behavior and race conditions between different users.

## Data Pipeline
The flow for this command is a simple, linear query pipeline. It reads from a world data source and transforms the data for two separate outputs: the server log and the command issuer's chat.

> Flow:
> User Command Input -> Server Command Dispatcher -> **PrefabPathListCommand.execute()** -> World Store (read WorldPathData) -> Data Transformation -> Output to Server Logger AND Output to Command Issuer

