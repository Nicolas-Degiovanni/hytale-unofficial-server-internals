---
description: Architectural reference for BlockInspectRotationCommand
---

# BlockInspectRotationCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block
**Type:** Transient

## Definition
```java
// Signature
public class BlockInspectRotationCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The BlockInspectRotationCommand is a server-side diagnostic tool designed for developers and administrators. Its primary function is to provide real-time, in-game visual feedback on the rotational state of blocks within the player's current chunk section.

This command serves as a bridge between three core server systems:
1.  **Command System:** It registers itself as an executable command, responding to player input.
2.  **World Storage System:** It queries the world's ChunkStore to asynchronously retrieve raw block and rotation data for a specific chunk section.
3.  **Debug Rendering System:** It injects temporary visual markers (colored cubes) into the world via DebugUtils to represent the orientation of blocks.

The command's design is fundamentally asynchronous. By using CompletableFuture, it avoids blocking the main server thread while waiting for potentially slow I/O operations related to chunk loading. This ensures that server performance is not degraded when this diagnostic tool is used.

## Lifecycle & Ownership
-   **Creation:** A single instance of BlockInspectRotationCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is registered under the alias *inspectrotation*.
-   **Scope:** The object instance is a singleton managed by the command registry and persists for the entire server session. However, each execution of the command is a distinct, short-lived operation.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is stateless. It contains no mutable instance fields. All data required for an operation, such as the player's location and the world context, is provided as arguments to the execute method. The static Message fields are immutable constants.

-   **Thread Safety:** The class instance is inherently thread-safe due to its stateless design. The core logic within the execute method is carefully managed for concurrency. The call to getChunkSectionReferenceAsync initiates an asynchronous operation. The subsequent processing logic within the thenAcceptAsync lambda is scheduled by the world's thread pool, guaranteeing safe access to chunk data without requiring explicit locks.

    **Warning:** The execute method is not designed to be called from arbitrary threads. It must be invoked by the server's Command System, which provides the necessary execution context and threading guarantees.

## API Surface
The public contract is defined by its role as a command. Direct programmatic invocation is not a standard use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | Asynchronously inspects all N blocks (typically 32768) in the invoking player's current chunk section. For each block with a non-default rotation, it renders a temporary debug cube. |

## Integration Patterns

### Standard Usage
This command is not intended for programmatic use. It is designed to be invoked by a privileged user through the in-game chat console.

> Example Player Input:
> `/inspectrotation`

The server's Command System is responsible for parsing this input, verifying player permissions, and dispatching the call to the singleton instance of this class.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BlockInspectRotationCommand()`. The server's command registry manages the lifecycle of command objects. Manually creating an instance will result in an un-registered and non-functional command.
-   **Programmatic Invocation:** Directly calling the execute method from other game logic bypasses the essential command processing pipeline, including permission checks and context setup. This is an unsupported and unstable integration path.
-   **Assuming Synchronicity:** Do not write code that depends on the command's visual effects being present immediately after invocation. The entire operation is asynchronous, from chunk loading to rendering, and may complete several ticks after the command is issued.

## Data Pipeline
The flow of data for this command begins with player input and results in both a chat message and a visual change in the world.

> Flow:
> Player Chat Input (`/inspectrotation`) -> Command Parser -> **BlockInspectRotationCommand.execute()** -> World ChunkStore (Async Request) -> Chunk Data -> Block Iteration & Rotation Check -> DebugUtils.addCube() -> Debug Rendering System -> Visual Feedback to Player
>
> Parallel Flow:
> **BlockInspectRotationCommand.execute()** -> PlayerRef.sendMessage() -> Network Layer -> "Done" Chat Message to Player

