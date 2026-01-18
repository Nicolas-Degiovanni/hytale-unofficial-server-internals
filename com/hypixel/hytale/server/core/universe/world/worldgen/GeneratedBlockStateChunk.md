---
description: Architectural reference for GeneratedBlockStateChunk
---

# GeneratedBlockStateChunk

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen
**Type:** Transient

## Definition
```java
// Signature
public class GeneratedBlockStateChunk {
```

## Architecture & Concepts
The GeneratedBlockStateChunk serves as a transient, in-memory data structure used exclusively during the server-side world generation pipeline. It functions as a mutable *staging area* or *builder* for the block data within a single vertical chunk column.

Its primary architectural role is to accumulate the output of various procedural generation passes, such as terrain shaping, ore vein placement, and vegetation spawning. Each pass modifies the state of the GeneratedBlockStateChunk instance for a given chunk column.

Once all generation passes for a column are complete, this object is converted into a BlockComponentChunk. The BlockComponentChunk is the canonical, runtime-ready component used by the live game engine to represent block data. This two-step process separates the highly mutable, write-heavy generation phase from the more read-optimized, stable state of the world in the game loop.

### Lifecycle & Ownership
The lifecycle of a GeneratedBlockStateChunk is ephemeral and tightly scoped to a single, atomic world generation task.

-   **Creation:** A new instance is instantiated by a world generator or a specific generation pipeline stage at the beginning of processing for a new chunk column. It is never pre-allocated or reused.
-   **Scope:** The object exists only for the duration of the generation process for one chunk column. It accumulates state as generator passes are executed.
-   **Destruction:** The object is intended to be immediately discarded and made eligible for garbage collection after its data is converted via the toBlockComponentChunk method. The resulting BlockComponentChunk assumes ownership of the finalized block data. Holding a reference to a GeneratedBlockStateChunk after this conversion is a memory leak anti-pattern.

## Internal State & Concurrency
-   **State:** The internal state is highly mutable, centered around a sparse map that stores block states against a packed coordinate index. This design is optimized for the iterative and often non-linear nature of procedural generation, where blocks are frequently set, overwritten, or cleared.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. The internal use of Int2ObjectOpenHashMap without any synchronization primitives is intentional. The world generation framework guarantees that the generation of a single chunk column is a single-threaded operation. Each worker thread processing a chunk column will operate on its own private instance of GeneratedBlockStateChunk.

## API Surface
The public API is minimal, focusing exclusively on the build-and-convert lifecycle.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getState(x, y, z) | Holder<ChunkStore> | O(1) | Retrieves the currently staged block state at the given local coordinates. |
| setState(x, y, z, state) | void | O(1) | Sets or clears the block state at the given local coordinates. This is the primary mutation method. |
| toBlockComponentChunk() | BlockComponentChunk | O(1) | Consumes the internal state to produce a finalized, runtime-ready BlockComponentChunk. This is a terminal operation. |

## Integration Patterns

### Standard Usage
The class is designed to be used in a straightforward create, populate, and convert sequence within a world generation task.

```java
// 1. A new instance is created for a specific chunk column task.
GeneratedBlockStateChunk stagingChunk = new GeneratedBlockStateChunk();

// 2. Multiple generation passes populate the staging object.
terrainGenerator.apply(stagingChunk);
oreGenerator.apply(stagingChunk);
vegetationGenerator.apply(stagingChunk);

// 3. The final data is converted into a runtime component.
// The stagingChunk is now eligible for garbage collection.
BlockComponentChunk finalChunkData = stagingChunk.toBlockComponentChunk();
world.getChunkStore().storeComponent(chunkCoords, finalChunkData);
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never access a single GeneratedBlockStateChunk instance from multiple threads. This will lead to data corruption and unpredictable generation results.
-   **Long-Term Storage:** Do not store instances of this class in caches or hold references to them after the call to toBlockComponentChunk. They are temporary builders and are not optimized for long-term storage or read access.
-   **Instance Re-use:** Do not attempt to clear and re-use an instance for a new chunk column. The design contract expects a fresh instance for each distinct generation task to ensure a clean state.

## Data Pipeline
GeneratedBlockStateChunk is a critical intermediate step in the flow of data from procedural noise functions to the persistent world state.

> Flow:
> World Generation Task -> **GeneratedBlockStateChunk** (Accumulation) -> toBlockComponentChunk() -> BlockComponentChunk -> ChunkStore -> Live Game World

---

