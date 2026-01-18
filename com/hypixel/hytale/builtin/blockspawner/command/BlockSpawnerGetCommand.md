---
description: Architectural reference for BlockSpawnerGetCommand
---

# BlockSpawnerGetCommand

**Package:** com.hypixel.hytale.builtin.blockspawner.command
**Type:** Singleton

## Definition
```java
// Signature
public class BlockSpawnerGetCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The **BlockSpawnerGetCommand** is a server-side command processor that implements the Command design pattern. It serves as a user-facing entry point for querying the state of a **BlockSpawner** component within the game world. This class is not a service or a manager; it is a stateless, single-purpose handler registered with the server's central **CommandSystem**.

Its primary architectural function is to decouple the raw user input (a chat or console command) from the complex, multi-step process of accessing world data. It translates a high-level user request into a series of low-level lookups against the world's component storage systems.

The command supports two modes of operation for target selection:
1.  **Explicit Coordinates:** The user provides a specific X, Y, Z world position.
2.  **Player Targeting:** If no coordinates are provided, the system uses the command sender's line of sight to determine the target block. This mode is only available to player-controlled entities.

## Lifecycle & Ownership
- **Creation:** A single instance of **BlockSpawnerGetCommand** is instantiated by the **CommandSystem** during server initialization or when its parent module is loaded. It is registered under the name "get" within the "blockspawner" command group.
- **Scope:** The object instance persists for the entire server session. It is a long-lived singleton managed by the command registry.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server shuts down or its defining module is unloaded, at which point it is removed from the **CommandSystem** registry.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only instance field, **positionArg**, is an immutable argument definition object initialized in the constructor. All state required for an operation is passed into the **execute** method via the **CommandContext** parameter, ensuring that each command execution is independent and isolated.

- **Thread Safety:** The **BlockSpawnerGetCommand** instance itself is thread-safe due to its stateless nature. However, the **execute** method is designed to be called exclusively from the main server thread as part of the game loop's command processing phase. It performs read operations on shared, mutable world state (e.g., **ChunkStore**).

    **WARNING:** Calling the **execute** method from an asynchronous thread will bypass engine-level concurrency controls and will lead to race conditions, data corruption, or server crashes. All interactions with world components must be synchronized with the main server tick.

## API Surface
The public API is defined by its parent, **AbstractWorldCommand**. The core logic is contained within the overridden **execute** method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | Variable | Executes the command logic. Complexity is variable; O(1) if the target chunk is in memory, but can incur significant I/O latency if the chunk must be loaded from disk. Throws **GeneralCommandException** on invalid input or if a target cannot be resolved. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic invocation. It is designed to be executed exclusively by the server's **CommandSystem** in response to user input. A user or a server administrator would trigger it via the console or chat.

*Example user input:*
```
/blockspawner get
```
*Example user input with coordinates:*
```
/blockspawner get 120 64 -250
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockSpawnerGetCommand()`. The command system manages the lifecycle of command objects. Creating a new instance has no effect as it will not be registered to handle any input.
- **Direct Invocation:** Never call the **execute** method directly. Doing so bypasses critical infrastructure, including argument parsing, permission checks, and context setup provided by the **CommandSystem**. This can lead to unpredictable behavior and **NullPointerExceptions**.
- **Stateful Modification:** Do not attempt to modify this class to hold state between executions. Command objects must be stateless to ensure predictable and isolated behavior for every user invocation.

## Data Pipeline
The flow of data for a single execution is a multi-stage read-only process that traverses the core world data structures.

> Flow:
> User Input String -> **CommandSystem** Parser -> **BlockSpawnerGetCommand**.execute() -> Target Resolution (Arguments or **TargetUtil**) -> **World**.getChunkStore() -> **ChunkStore**.getChunkReference() -> **WorldChunk**.getBlockComponentEntity() -> **ChunkStore**.getComponent(**BlockSpawner**.class) -> Read **blockSpawnerId** -> **Message** Factory -> **CommandContext**.sendMessage() -> Network Packet to Client

