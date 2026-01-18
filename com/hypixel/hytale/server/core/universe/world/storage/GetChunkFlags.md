---
description: Architectural reference for GetChunkFlags
---

# GetChunkFlags

**Package:** com.hypixel.hytale.server.core.universe.world.storage
**Type:** Utility

## Definition
```java
// Signature
public class GetChunkFlags {
```

## Architecture & Concepts
GetChunkFlags is a static utility class that defines a contract for controlling the server's chunk retrieval process. It is not a system that performs work, but rather a set of bitmask flags that configure the behavior of the primary chunk management and storage systems.

The core design principle is to provide a flexible and extensible mechanism for requesting world chunks without creating a rigid and bloated API on the chunk provider. Instead of methods like `getChunkWithoutGenerating` or `getChunkAndSetTicking`, a single `getChunk` method accepts an integer composed of these flags. This allows callers, such as the entity simulation or world generation systems, to specify their exact requirements with fine-grained control.

This class acts as a configuration object passed by value, decoupling the requester of a chunk from the complex internal logic of the chunk cache, storage engine, and world generator.

### Lifecycle & Ownership
- **Creation:** As a class containing only static final fields, GetChunkFlags is never instantiated. Its constants are initialized by the JVM during class loading.
- **Scope:** The flags are application-scoped and available globally once the class is loaded. They are immutable compile-time constants.
- **Destruction:** The class and its static members are unloaded from memory when the server process terminates.

## Internal State & Concurrency
- **State:** GetChunkFlags is entirely stateless. All fields are `public static final int`, making them immutable constants.
- **Thread Safety:** The class is inherently thread-safe. Its constant values can be read from any thread without risk of race conditions or memory visibility issues. The integer flags themselves are atomic.

## API Surface
The public contract consists exclusively of integer constants intended for use as a bitmask.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NONE | int | N/A | Represents the default behavior. The system will use its standard logic to load from disk or generate a new chunk if necessary. |
| NO_LOAD | int | N/A | **Critical:** Prevents a disk read. If the requested chunk is not in the in-memory cache, the request will fail or return null instead of loading from storage. |
| NO_GENERATE | int | N/A | Prevents the world generator from being invoked. If the chunk is not in memory or on disk, the request will fail. |
| SET_TICKING | int | N/A | Instructs the chunk manager to transition the chunk to a "ticking" state upon successful retrieval, enabling entity processing and random block ticks. |
| BYPASS_LOADED | int | N/A | **Warning:** Forces the retrieval logic to bypass the primary in-memory cache. This is a specialized flag for debugging or cache invalidation and should not be used in standard game logic. |
| POLL_STILL_NEEDED | int | N/A | Signals that the request is a low-priority poll to check if a chunk is still required by any system, often as part of a chunk unloading or garbage collection heuristic. |
| NO_SET_TICKING_SYNC | int | N/A | A special modifier, not a standard bit flag. It alters the behavior of SET_TICKING to prevent a synchronous state change, likely deferring the transition to a later stage in the game tick. |

## Integration Patterns

### Standard Usage
Flags are combined using the bitwise OR operator to create a composite request configuration. The resulting integer is passed to the chunk retrieval system.

```java
// Example: Request a chunk for an AI entity that needs to pathfind.
// The chunk must be fully active (ticking) for simulation.
// If the chunk doesn't exist at all, we do not want to trigger world-gen.
int retrievalFlags = GetChunkFlags.SET_TICKING | GetChunkFlags.NO_GENERATE;

// The World or ChunkManager service consumes the flags.
Chunk targetChunk = server.getWorld().getChunk(chunkPosition, retrievalFlags);

if (targetChunk != null) {
    // Proceed with pathfinding...
}
```

### Anti-Patterns (Do NOT do this)
- **Incorrect Flag Checking:** Do not use the equality operator `==` to check for a flag. This will fail if multiple flags are combined. Always use a bitwise AND operation.
  ```java
  // INCORRECT
  if (flags == GetChunkFlags.NO_LOAD) { ... }

  // CORRECT
  if ((flags & GetChunkFlags.NO_LOAD) != 0) { ... }
  ```
- **Arithmetic Combination:** Do not use arithmetic operators like `+` to combine flags. This can produce incorrect and unpredictable results. Always use bitwise OR `|`.

## Data Pipeline
GetChunkFlags does not process data itself; it directs the flow of a data request within the world system.

> Flow:
> Game System (e.g., Player Logic) -> Creates `int flags` using **GetChunkFlags** -> `World.getChunk(pos, flags)` -> ChunkManager -> **Flag Interpretation Logic** -> (Conditional) Chunk Cache / Chunk Storage IO / World Generator -> Return `Chunk` object

