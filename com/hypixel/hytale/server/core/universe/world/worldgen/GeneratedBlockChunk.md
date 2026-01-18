---
description: Architectural reference for GeneratedBlockChunk
---

# GeneratedBlockChunk

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen
**Type:** Transient

## Definition
```java
// Signature
public class GeneratedBlockChunk {
```

## Architecture & Concepts

The GeneratedBlockChunk is a fundamental component of the server-side world generation pipeline. It serves as a **mutable, in-memory representation** of a world chunk during its creation phase. This class is not the final data structure used by the live game simulation; rather, it is a high-level, unoptimized staging area.

Its primary architectural role is to act as a **Builder** for a chunk. Various world generation stages—such as terrain shaping, cave carving, ore placement, and structure spawning—sequentially operate on a single GeneratedBlockChunk instance. Each stage populates or modifies its block data, environment data, and biome tints.

Once all generation stages are complete, this object is transformed into the highly optimized, network-ready BlockChunk format via the `toBlockChunk` method. This design decouples the complex, stateful process of world generation from the efficient, read-optimized representation required by the running game server.

### Lifecycle & Ownership

-   **Creation:** A GeneratedBlockChunk is instantiated by a world generator service at the beginning of the generation process for a specific chunk coordinate. It is a short-lived object.
-   **Scope:** The object's lifecycle is strictly confined to the execution of the world generation pipeline for a single chunk. It does not persist between server restarts or exist outside this context.
-   **Destruction:** After the `toBlockChunk` method is called and the resulting BlockChunk is committed to the ChunkStore, the GeneratedBlockChunk instance is no longer referenced and becomes eligible for garbage collection. It is designed to be discarded after use.

## Internal State & Concurrency

-   **State:** The state is entirely **mutable**. Its core purpose is to accumulate changes from multiple, sequential generation algorithms. Data is stored in an array of `GeneratedChunkSection` objects, which are lazily instantiated to conserve memory for vertically empty chunks.
-   **Thread Safety:** This class is **not thread-safe**. It is designed for exclusive use by a single worker thread responsible for generating a specific chunk. Any concurrent access from multiple threads will lead to race conditions, data corruption, and non-deterministic world generation. All operations on an instance must be externally synchronized, typically by the world generation job scheduler which assigns one chunk task per thread.

## API Surface

The public API is designed for mutation during generation and a final conversion.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setBlock(x, y, z, blockId, rotation, filler) | void | O(1) | The primary mutation method. Places a block at the specified coordinates. Lazily initializes the vertical chunk section if it does not exist. |
| getHeight(x, z) | int | O(H) | Calculates the highest non-transparent block at a column. **Warning:** This is a potentially expensive operation as it scans vertically from the world ceiling. |
| generateHeight() | ShortBytePalette | O(W\*D\*H) | Generates a complete heightmap for the chunk. **Warning:** This is a very expensive operation and should only be called during the finalization step. |
| toBlockChunk(sectionHolders) | BlockChunk | O(N + W\*D\*H) | Finalizes the chunk. Converts the internal mutable state into an immutable, game-ready BlockChunk. This triggers the expensive `generateHeight` call. |
| setEnvironment(x, y, z, environment) | void | O(1) | Sets environment data, such as for special cave biomes, at a specific block coordinate. |
| setTint(x, z, tint) | void | O(1) | Sets the biome tint color for a vertical column. |

## Integration Patterns

### Standard Usage

The canonical usage pattern involves a controller or generator service passing an instance through a chain of modifiers before finalizing it.

```java
// 1. A world generator creates an instance for a new chunk
GeneratedBlockChunk generatedChunk = new GeneratedBlockChunk(index, x, z);

// 2. It is passed through sequential generation stages
terrainGenerator.apply(generatedChunk);
caveGenerator.apply(generatedChunk);
oreGenerator.apply(generatedChunk);
structurePopulator.apply(generatedChunk);

// 3. The finalized data is extracted and stored
// The 'holders' array is prepared by the ChunkStore system.
BlockChunk finalChunk = generatedChunk.toBlockChunk(holders);
chunkStore.commit(finalChunk);

// The 'generatedChunk' instance is now discarded.
```

### Anti-Patterns (Do NOT do this)

-   **Long-Term Storage:** Do not hold references to a GeneratedBlockChunk after the generation process is complete. It is not optimized for memory and keeping it alive constitutes a memory leak.
-   **Concurrent Modification:** Never pass a single GeneratedBlockChunk instance to multiple threads for simultaneous modification. This will corrupt the chunk data. The world generation system must ensure one thread per chunk task.
-   **Re-use:** Do not attempt to clear and reuse a GeneratedBlockChunk for a different chunk coordinate. The internal state is not designed to be reset. Always create a new instance.
-   **Premature Optimization:** Avoid calling expensive methods like `getHeight` or `generateHeight` repeatedly during the generation stages. These should only be used during finalization or for specific lookups where performance is not critical.

## Data Pipeline

The GeneratedBlockChunk is a transient step in the data flow from procedural generation logic to persistent world storage.

> Flow:
> World Generator -> **GeneratedBlockChunk (Creation & Mutation)** -> Generator Stages (Apply Terrain, Caves, Ores) -> `toBlockChunk()` -> BlockChunk (Immutable Representation) -> ChunkStore (Persistence & Caching)

