---
description: Architectural reference for RelativeChunkPosition
---

# RelativeChunkPosition

**Package:** `com.hypixel.hytale.server.core.command.system.arguments.types`
**Type:** Transient

## Definition
```java
// Signature
public class RelativeChunkPosition {
```

## Architecture & Concepts

The RelativeChunkPosition class is a specialized data structure within the server's command system. It serves as a high-level representation of a two-dimensional chunk coordinate that can be either absolute (e.g., `10, 25`) or relative to a command sender's current position (e.g., `~, ~5`).

Its primary architectural role is to decouple command argument parsing from command execution. The command parser is responsible for interpreting the raw string input and constructing a RelativeChunkPosition instance. The command executor then uses this object to resolve the input into a concrete `Vector2i` world coordinate at the moment of execution. This class encapsulates the logic for handling tilde notation (`~`), abstracting the complexity of resolving coordinates based on an entity's `TransformComponent`.

This is not a long-lived service or component, but rather a short-lived value object that carries parsed command data through the execution pipeline.

### Lifecycle & Ownership
-   **Creation:** An instance is created by a command argument parser, specifically one designed to handle chunk coordinates, during the parsing phase of a command. It is constructed with two `IntCoord` objects, which themselves represent the parsed values for the X and Z axes.
-   **Scope:** The object's lifetime is strictly limited to the duration of a single command's processing cycle. It is typically held as a parsed argument within the `CommandContext`.
-   **Destruction:** It is eligible for garbage collection immediately after the command has finished executing and the `CommandContext` is discarded. There are no external references or cleanup procedures.

## Internal State & Concurrency
-   **State:** The class is **immutable**. Its internal state consists of two final `IntCoord` fields, `x` and `z`. Once the object is constructed, its coordinate representation cannot be changed.
-   **Thread Safety:** RelativeChunkPosition is inherently **thread-safe** due to its immutability. It can be safely passed between threads. However, its resolution methods, such as `getChunkPosition`, depend on external and potentially mutable state from the `CommandContext` and `ComponentAccessor`.

    **Warning:** While the object itself is safe, calling its resolution methods from multiple threads without proper synchronization of the underlying world state (e.g., the entity's position) will lead to race conditions and unpredictable results. The command system is designed to execute commands on the main server thread, making this a non-issue in standard usage.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChunkPosition(context, accessor) | Vector2i | O(1) | Resolves the position into a concrete chunk coordinate based on the command sender's location. Throws `GeneralCommandException` if the sender is not a player in a world. |
| getChunkPosition(base) | Vector2i | O(1) | Resolves the position relative to an explicitly provided `Vector3d` base coordinate. |
| isRelative() | boolean | O(1) | Returns true if either the X or Z coordinate is relative (uses tilde notation). |

## Integration Patterns

### Standard Usage
This class is not intended to be instantiated directly by developers. It is received as a parsed argument within a command's execution logic.

```java
// Inside a command's run() method:

// 1. Retrieve the parsed argument from the context
RelativeChunkPosition targetChunk = context.getArgument("destination", RelativeChunkPosition.class);

// 2. Resolve it to a concrete coordinate vector
// The ComponentAccessor is typically provided to the command executor.
Vector2i chunkCoords = targetChunk.getChunkPosition(context, entityStoreAccessor);

// 3. Use the resolved coordinates for game logic
world.loadChunk(chunkCoords);
```

### Anti-Patterns (Do NOT do this)
-   **Caching Instances:** Do not store an instance of RelativeChunkPosition for later use. A relative position like `~ ~` is only valid at the exact moment of execution. Caching and re-using it later will resolve against the sender's *new* position, leading to incorrect behavior.
-   **Ignoring Sender Type:** Calling `getChunkPosition(context, ...)` for a command sent from the server console will fail, as there is no player entity to resolve relative coordinates against. Always check the command sender or use `isRelative()` to validate that a non-player is not attempting to use relative coordinates.

## Data Pipeline
The class acts as an intermediate container in the flow of data from player input to world action.

> Flow:
> Raw Command String (`/teleportchunk ~5 ~`) -> Command Argument Parser -> **RelativeChunkPosition** (instance created) -> Command Executor -> `getChunkPosition()` call -> `Vector2i` -> World System (e.g., ChunkManager)

