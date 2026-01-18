---
description: Architectural reference for PrefabPathMergeCommand
---

# PrefabPathMergeCommand

**Package:** com.hypixel.hytale.builtin.path.commands
**Type:** Transient

## Definition
```java
// Signature
public class PrefabPathMergeCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The PrefabPathMergeCommand is a server-side command handler responsible for executing the logic to merge one prefab path into another. As a subclass of AbstractPlayerCommand, its execution is intrinsically tied to a player entity and is invoked by the server's central command processing system.

This class acts as an orchestrator, not a data container. Its primary function is to bridge player input with the underlying world data systems. It interacts with three critical components:

1.  **BuilderToolsPlugin State:** It retrieves the player's currently selected or "active" path. This establishes the destination for the merge operation.
2.  **CommandContext:** It parses the command arguments to identify the source path (the one to be merged and subsequently deleted).
3.  **WorldPathData:** This world-level resource is the source of truth for all path entities. The command fetches both the active and target IPrefabPath instances from this repository, performs the merge, and then instructs the repository to remove the source path.

The operation is destructive; the source path is consumed by the merge and permanently deleted from the world.

### Lifecycle & Ownership
-   **Creation:** A single instance of PrefabPathMergeCommand is instantiated by the server's command registration system during server startup or plugin loading. It is registered under the name "merge" within its parent command group.
-   **Scope:** The object instance persists for the entire server session, held within the command registry. However, its execution context is ephemeral, lasting only for the duration of a single invocation of the `execute` method.
-   **Destruction:** The instance is garbage collected when the server shuts down or when its parent plugin is unloaded, which clears the command registry.

## Internal State & Concurrency
-   **State:** The class is effectively stateless from an execution perspective. Its only member field, `targetPathIdArg`, is an immutable definition of a required command argument, configured once in the constructor. All data required for the `execute` method is passed in as parameters or retrieved from other stateful systems like WorldPathData.
-   **Thread Safety:** This class is **not thread-safe** and must only be accessed by the main server thread. It operates directly on shared, mutable game state (Store, WorldPathData) without any internal locking. It relies entirely on the server's single-threaded game loop to ensure safe execution and prevent race conditions.

## API Surface
The public API is defined by its role as a command. The primary entry point is the `execute` method, which is invoked by the framework, not by developers directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | Executes the merge logic. Complexity is proportional to the number of nodes (N) in the source path being merged. Throws GeneralCommandException if the player has no active path. |

## Integration Patterns

### Standard Usage
This class is not designed for direct invocation. The standard and only supported usage is through a player executing the command in-game. The server's command system handles parsing, context creation, and invocation.

A player triggers this command by typing:
`/path merge <target-path-uuid>`

The framework then invokes the command instance:
```java
// Conceptual representation of framework invocation
Command command = CommandRegistry.find("path merge");
CommandContext context = CommandParser.parse(playerInput, command);

if (command instanceof PrefabPathMergeCommand) {
    // The framework provides the necessary world and entity state
    ((PrefabPathMergeCommand) command).execute(context, server.getStore(), ...);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PrefabPathMergeCommand()`. The command system manages the lifecycle of command objects. Manual instantiation creates an unmanaged object that will not be reachable by players.
-   **Manual Execution:** Calling the `execute` method directly bypasses critical framework services like argument parsing, permission checks, and context setup. This can lead to unpredictable behavior and server instability.
-   **State Assumption:** Do not assume a path is loaded. The command's implementation correctly checks `isFullyLoaded` on both paths before attempting the merge. Any logic that bypasses these checks risks operating on incomplete or invalid data.

## Data Pipeline
The flow of data for a successful merge operation is linear and transactional, managed entirely within the `execute` method.

> Flow:
> Player Chat Input (`/path merge ...`) -> Server Command Parser -> **PrefabPathMergeCommand.execute()**
> 1.  Reads active path UUID from the player's BuilderToolsPlugin state.
> 2.  Reads target path UUID from the CommandContext.
> 3.  Fetches active IPrefabPath object from WorldPathData resource.
> 4.  Fetches target IPrefabPath object from WorldPathData resource.
> 5.  Invokes `targetPath.mergeInto(activePath)`.
> 6.  Invokes `worldPathData.removePrefabPath(targetPathId)`.
> 7.  Sends a confirmation Message back to the PlayerRef.

