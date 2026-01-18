---
description: Architectural reference for BlockInspectPhysicsCommand
---

# BlockInspectPhysicsCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block
**Type:** Transient

## Definition
```java
// Signature
public class BlockInspectPhysicsCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The BlockInspectPhysicsCommand is a server-side, player-executable diagnostic tool designed to visualize the internal block physics stability system. It functions as a transient bridge between the command system and the low-level chunk data storage, providing developers and administrators with a real-time view of structural support values within a specific world region.

Architecturally, this command is an endpoint in the server's command processing pipeline. Upon invocation, it performs the following critical operations:
1.  **Context Acquisition:** It identifies the executing player's location by querying the Entity-Component-System (ECS) for their TransformComponent.
2.  **Asynchronous Data Fetch:** It initiates a non-blocking request to the World's ChunkStore for the chunk section corresponding to the player's position. This is the most critical architectural choice, preventing the server's main thread from stalling on potentially slow disk I/O operations while loading chunk data.
3.  **Data Processing & Visualization:** Within the asynchronous callback, it accesses the BlockPhysics component of the loaded chunk. It iterates through the physics data, mapping support values to color-coded debug cubes.
4.  **Debug Rendering:** It leverages the DebugUtils service to dispatch rendering information to the client, effectively "drawing" the physics state in the world for the invoking player.

This command is a prime example of a read-only diagnostic that safely interacts with core world systems without mutation, using asynchronous patterns to maintain server performance.

### Lifecycle & Ownership
-   **Creation:** A single instance of BlockInspectPhysicsCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is registered against the command name "inspectphys".
-   **Scope:** The object instance persists for the entire lifetime of the server. However, each execution of the command is a distinct, short-lived operation. The state associated with an execution (e.g., player position, chunk data) is scoped exclusively to the `execute` method call and its subsequent asynchronous callbacks.
-   **Destruction:** The instance is garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The BlockInspectPhysicsCommand object is effectively stateless between invocations. Its only member field, `ALL`, is a final FlagArg used to define a command-line argument and is immutable after construction. All state required for execution is passed in via the `execute` method parameters or derived within the method's local scope.

-   **Thread Safety:** This class is thread-safe by design. The `execute` method is invoked by the server's command handler, typically on the main server thread. The most intensive work—loading and processing chunk data—is explicitly offloaded from the calling thread via `getChunkSectionReferenceAsync`. The subsequent processing logic within `thenAcceptAsync` is scheduled to run on the world's designated thread pool, ensuring safe, synchronized access to chunk and world data without manual locking.

    **Warning:** The asynchronous nature is fundamental. Any modification that blocks within the initial `execute` method call would introduce severe performance degradation.

## API Surface
The public contract is defined by its inheritance from AbstractPlayerCommand. The primary entry point is the `execute` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N) | Asynchronously inspects the chunk at the player's location and renders its physics data. N is the number of blocks in a chunk section (32768), but performance is dominated by the latency of chunk loading. |

## Integration Patterns

### Standard Usage
This command is not intended to be used programmatically. It is designed for invocation by a player or server administrator through the in-game chat console.

```
// Player types this command in chat
/inspectphys

// To view all blocks, not just those with physics data
/inspectphys all
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new BlockInspectPhysicsCommand()`. The command system manages the lifecycle of all command objects. Direct instantiation will result in a non-functional object that is not registered to handle any command strings.
-   **Manual Execution:** Do not call the `execute` method directly. This bypasses the entire command processing pipeline, including argument parsing, permission checks, and context injection. The method will fail with exceptions if its context parameters are not correctly populated by the command system.

## Data Pipeline
The flow of data for a single command execution is asynchronous and crosses several major system boundaries.

> Flow:
> Player Chat Input (`/inspectphys`) -> Server Network Layer -> Command Parser -> **BlockInspectPhysicsCommand.execute** -> Asynchronous request to ChunkStore -> Chunk Data (from disk/cache) -> **BlockInspectPhysicsCommand callback** -> Iteration over BlockPhysics data -> DebugUtils.addCube -> Debug Render Packet -> Server Network Layer -> Client Rendering Engine

