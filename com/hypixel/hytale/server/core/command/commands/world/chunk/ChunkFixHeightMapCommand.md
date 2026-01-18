---
description: Architectural reference for ChunkFixHeightMapCommand
---

# ChunkFixHeightMapCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient

## Definition
```java
// Signature
public class ChunkFixHeightMapCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The ChunkFixHeightMapCommand is a server-side administrative tool designed for world maintenance and debugging. It provides a direct interface to correct inconsistencies in a chunk's heightmap data, a critical optimization structure used by multiple core engine systems.

Architecturally, this class serves as a command-line entry point into the world's low-level data structures. It directly interacts with the `ChunkStore` to retrieve chunk components and the `ChunkLightingManager` to trigger complex, asynchronous recalculations. Its primary function is to invoke the `BlockChunk.updateHeightmap` method, which scans every block column in the chunk to determine the highest non-air block, and then to manage the subsequent lighting invalidation and recalculation process.

The command's design acknowledges the performance-intensive nature of lighting updates. Rather than blocking the main server thread, it initiates the lighting recalculation and then schedules a separate, periodic task to poll for its completion. This ensures server responsiveness is maintained while the expensive operation completes in the background.

### Lifecycle & Ownership
- **Creation:** A single instance is created by the server's command registration system during the server bootstrap sequence.
- **Scope:** The command object is a stateless singleton that persists for the entire server session, held within the central command registry. The execution logic, however, is transient and scoped to a single invocation of the command.
- **Destruction:** The object is garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its fields, such as `chunkPosArg`, are initialized in the constructor and are not mutated during execution. All stateful operations are performed on objects passed into the `execute` method, primarily the `World` and its constituent components.

- **Thread Safety:** The `execute` method is invoked on the server's main thread by the command system. The class is not designed for concurrent invocations. The most critical concurrency consideration is its interaction with the `ChunkLightingManager`. The command safely initiates a background lighting task and uses a non-blocking polling mechanism via the `HytaleServer.SCHEDULED_EXECUTOR`. This prevents the main server thread from stalling while waiting for the lighting engine. The final notification to clients is dispatched through the `world.getNotificationHandler()`, which is responsible for managing thread-safe updates.

## API Surface
The public contract is defined by its parent, `AbstractWorldCommand`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(N*M) + Async | The primary entry point. N is chunk height, M is chunk area. Triggers a full heightmap scan and an asynchronous lighting recalculation for the specified chunk. Sends feedback messages to the command context. |

## Integration Patterns

### Standard Usage
This command is intended to be invoked by a server administrator or a player with appropriate permissions via the server console or in-game chat. The command system handles parsing, argument resolution, and invocation.

```java
// This code is conceptual. In practice, a user types the command.
// Command issued by user: /chunk fixheight 10 20

// The server's CommandSystem would internally perform an action similar to:
CommandContext context = ...; // Context for the user who ran the command
World world = ...; // The world the user is in
Store<EntityStore> store = ...; // The entity store for the world

// The command system finds the registered instance and executes it
ChunkFixHeightMapCommand command = commandRegistry.get("fixheight");
command.execute(context, world, store);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ChunkFixHeightMapCommand()`. The command is useless unless registered with the server's command system, which handles its lifecycle.
- **Manual Invocation:** Avoid calling the `execute` method directly from other game logic. This command is a high-level administrative tool, not a general-purpose API for world modification. For programmatic world changes, use the appropriate world or chunk APIs directly.
- **Ignoring Asynchronicity:** The command's effects, particularly lighting, are not instantaneous. Logic that depends on the heightmap and lighting being fully corrected must wait for the `...lightingFinished` message or implement a similar callback/polling mechanism.

## Data Pipeline
This command initiates a complex control and data flow rather than processing a continuous stream of data.

> Flow:
> User Input (`/chunk fixheight ...`) -> Command System Parser -> **ChunkFixHeightMapCommand.execute()** -> ChunkStore (Read `BlockChunk`) -> `BlockChunk.updateHeightmap()` (Mutate) -> `ChunkLightingManager.invalidateLightInChunk()` (Trigger Async Task) -> Scheduled Executor (Poll for completion) -> `NotificationHandler.updateChunk()` -> Network Packet (To Clients)

