---
description: Architectural reference for OverridableChunkAccessor
---

# OverridableChunkAccessor

**Package:** com.hypixel.hytale.server.core.universe.world.accessor
**Type:** Contract Interface

## Definition
```java
// Signature
public interface OverridableChunkAccessor<X extends BlockAccessor> extends ChunkAccessor<X> {
```

## Architecture & Concepts
The OverridableChunkAccessor interface defines a contract for components that require the ability to perform a complete, destructive replacement of a world chunk's block data. It extends the read-only contract of ChunkAccessor, adding a single, high-impact write operation: `overwrite`.

This interface represents a powerful capability within the server's world data management system. It is the primary mechanism for systems like world generators, region file loaders, or large-scale world editing tools to apply bulk changes to the game universe. Unlike granular block modification APIs, this interface is designed for efficiency in scenarios where an entire chunk's state is being replaced at once.

Its existence implies a clear separation of concerns in the engine:
1.  **Read-only access:** Provided by the base ChunkAccessor for most game logic (e.g., physics, rendering data preparation, AI pathfinding).
2.  **Mutable access:** Provided by OverridableChunkAccessor for specialized, authoritative systems that control the fundamental state of the world.

## Lifecycle & Ownership
As an interface, OverridableChunkAccessor does not have its own lifecycle. Instead, it defines the expected lifecycle for any implementing class.

-   **Creation:** Implementations are typically created on-demand by a higher-level world management service. They are often short-lived, representing a transactional handle to a specific chunk for the duration of a single, atomic operation. For example, a WorldGenerator might request an OverridableChunkAccessor for chunk (X, Y) to apply newly generated terrain data.
-   **Scope:** The scope of an implementing object should be considered extremely narrow. It is a temporary handle and must not be cached or held for longer than the immediate overwrite operation. Holding a reference can lead to stale data views or prevent the chunk from being unloaded.
-   **Destruction:** The object is typically eligible for garbage collection immediately after the `overwrite` method is called and the reference goes out of scope. There is no explicit destruction or cleanup method defined in the contract.

## Internal State & Concurrency
-   **State:** The interface itself is stateless. However, any class implementing OverridableChunkAccessor is inherently stateful, as it holds a reference or handle to the underlying, mutable chunk data.
-   **Thread Safety:** **Not thread-safe.** This contract provides no guarantees of concurrent access safety. The `overwrite` method is a high-contention operation that performs a bulk data replacement. It is the responsibility of the calling system to ensure that no other thread is reading from or writing to the target chunk while an overwrite is in progress. Implementations are expected to be accessed only by the world generation or main server thread within a properly synchronized context.

**WARNING:** Concurrent modification of a chunk via this interface without external locking will lead to world corruption, server instability, and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| overwrite(X blockAccessor) | void | O(N) | Replaces the entire block data of the chunk with the data provided by the given BlockAccessor. N is the number of blocks in a chunk. This is a destructive, atomic operation. |

## Integration Patterns

### Standard Usage
The typical pattern involves a system (like a world generator) preparing a complete set of block data in a separate BlockAccessor implementation and then using an OverridableChunkAccessor to commit that data to the live world state.

```java
// How a developer should normally use this
// Assume 'world' is a service that can provide accessors
// Assume 'generatorResult' is a BlockAccessor containing new terrain data

// 1. Request a mutable accessor for a specific chunk
OverridableChunkAccessor accessor = world.getOverridableChunkAccessor(chunkPos);

// 2. Perform the complete data replacement
accessor.overwrite(generatorResult);

// 3. The accessor reference should now be discarded
```

### Anti-Patterns (Do NOT do this)
-   **Long-Lived References:** Do not store a reference to an OverridableChunkAccessor in a field or cache. It represents a temporary, transactional lock on the chunk and must be released immediately after use.
-   **Concurrent Writes:** Never share an OverridableChunkAccessor instance across multiple threads. Do not attempt to call `overwrite` from a different thread than the one that acquired the accessor without explicit, engine-provided synchronization mechanisms.
-   **Granular Modification:** Do not use this interface to change a single block. It is designed for bulk replacement, and using it for small changes is highly inefficient and semantically incorrect. Use a dedicated block modification API for that purpose.

## Data Pipeline
This interface is a terminal endpoint in many data generation pipelines. It acts as the bridge between an in-memory representation of a chunk and the live, in-game world state.

> Flow:
> Procedural Noise Generator -> Biome Placer -> Structure Populator -> In-Memory **BlockAccessor** -> **OverridableChunkAccessor**.overwrite() -> Live World State

