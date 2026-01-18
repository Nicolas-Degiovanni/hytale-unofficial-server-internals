---
description: Architectural reference for PrefabEditUpdateBoxCommand
---

# PrefabEditUpdateBoxCommand

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.commands
**Type:** Stateless Singleton Handler

## Definition
```java
// Signature
public class PrefabEditUpdateBoxCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts

The PrefabEditUpdateBoxCommand class is a command handler responsible for processing a player's request to redefine the bounding box of a prefab they are currently editing. It serves as a transactional controller within the server's command processing framework, specifically for the Builder Tools plugin.

Its primary architectural role is to act as the bridge between a player's direct input (a chat command) and the stateful prefab editing system. It validates the player's context, interprets their in-world selection, enforces safety rules, and orchestrates the modification of the underlying PrefabEditSession state.

A key design feature is its safety mechanism for anchor point relocation. If the new bounding box does not contain the prefab's existing anchor, the command will fail with a warning unless the player explicitly provides a confirmation flag. This prevents accidental and destructive changes to a prefab's origin.

Furthermore, the command defers the actual data modification to a separate work queue managed by the BuilderToolsPlugin. This is a critical concurrency pattern that prevents the command execution from blocking the main server thread with potentially expensive operations.

### Lifecycle & Ownership
- **Creation:** A single instance of PrefabEditUpdateBoxCommand is created and registered with the server's command system when the BuilderToolsPlugin is initialized. It is not instantiated per-execution.
- **Scope:** The singleton instance persists for the entire lifecycle of the server process, or until the parent plugin is unloaded.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the BuilderToolsPlugin is disabled or the server shuts down.

## Internal State & Concurrency
- **State:** This class is fundamentally stateless. Its only instance field, confirmAnchorDeletionArg, is an immutable definition of a command-line flag. All state required for execution (player, world, selection) is passed into the execute method or retrieved from external state managers like PrefabEditSessionManager.

- **Thread Safety:** The execute method is invoked by the server's command processing thread and is **not thread-safe** for concurrent invocation. The engine's command system guarantees serialized execution, mitigating this risk. To maintain server performance, this class offloads the final state modification to a dedicated work queue, ensuring the main thread remains unblocked.

## API Surface

The primary contract is the overridden execute method, which is invoked by the command system, not by developers directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(1) | Processes the command. Validates player state, checks for anchor movement, and queues the final update. Throws no checked exceptions. |
| isLocationWithinSelection(location, selection) | boolean | O(1) | A pure utility function to check if a 3D point is within a given selection volume. |

## Integration Patterns

### Standard Usage

Developers do not instantiate or call this class directly. Interaction occurs when a player with appropriate permissions executes the corresponding command in-game, typically `/editprefab setbox`. The server's command framework routes the request to this handler.

The following example illustrates the *consequence* of a player executing the command, which triggers a state update within their session.

```java
// Conceptual flow triggered by an in-game command
// 1. Player makes a selection in the world and types the command.

// 2. The system invokes the handler, which accesses the session manager.
PrefabEditSessionManager manager = BuilderToolsPlugin.get().getPrefabEditSessionManager();
PrefabEditSession session = manager.getPrefabEditSession(playerUUID);

// 3. The handler's logic queues an update.
// This is the core operation performed by the command.
session.updatePrefabBounds(prefabUUID, newSelectionMin, newSelectionMax);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PrefabEditUpdateBoxCommand()`. The object is useless unless registered with the server's command framework, which is handled by the BuilderToolsPlugin.
- **Manual Invocation:** Never call the `execute` method directly. It depends on a fully-formed CommandContext and valid world state that is only supplied by the command processing pipeline. Bypassing the pipeline will lead to NullPointerExceptions and invalid state.
- **Blocking Operations:** Do not modify this class to perform file I/O, heavy computations, or other long-running tasks directly within the `execute` method. Adhere to the existing pattern of deferring work to the BuilderToolsPlugin queue.

## Data Pipeline

The command processes data flowing from player input through various server systems to ultimately modify persistent prefab data.

> Flow:
> Player Chat Input (`/setbox --confirm`) -> Server Command Parser -> **PrefabEditUpdateBoxCommand.execute()** -> Validation & State Check -> BuilderToolsPlugin Work Queue -> PrefabEditSession.updatePrefabBounds -> Prefab State Mutation

