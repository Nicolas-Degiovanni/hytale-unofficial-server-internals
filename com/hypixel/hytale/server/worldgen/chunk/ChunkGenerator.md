---
description: Architectural reference for ChunkGenerator
---

# ChunkGenerator

**Package:** com.hypixel.hytale.server.worldgen.chunk
**Type:** Service

## Definition
```java
// Signature
public class ChunkGenerator implements IBenchmarkableWorldGen, ValidatableWorldGen, MetricProvider, IWorldMapProvider {
```

## Architecture & Concepts
The ChunkGenerator is the central orchestrator for all procedural world generation on the server. It serves as the high-level entry point for creating terrain, placing biomes, carving caves, and populating the world with structures and entities. Its primary responsibility is to transform a world seed and a set of coordinates into a fully realized, game-ready chunk of the world.

Architecturally, this class is a stateful, asynchronous, and heavily cached service. It is not a simple collection of static utility methods; it is a complex system designed for high-throughput, parallel chunk generation.

Key architectural pillars include:

*   **Asynchronous Execution:** All chunk generation operations are executed on a dedicated internal thread pool, managed by a ChunkThreadPoolExecutor. The primary `generate` method returns a CompletableFuture, ensuring that the calling thread (typically the main server thread) is not blocked by computationally expensive world generation tasks.
*   **Multi-Layered Caching:** Performance is critically dependent on an aggressive caching strategy. The ChunkGenerator maintains several caches for intermediate generation data, including Zone and Biome information (ChunkGeneratorCache), cave structures (CaveGeneratorCache), and unique, world-specific prefabs (UniquePrefabCache). This prevents redundant calculations for adjacent chunks that share underlying data.
*   **Thread-Local Resources:** To manage concurrency and avoid contention on shared resources like Random number generators, the class utilizes a ThreadLocal variable. The static `getResource` method provides each worker thread with its own isolated ChunkGeneratorResource instance, a crucial pattern for scalable parallel processing.
*   **Composition over Inheritance:** The generator composes various specialized components, such as the ZonePatternProvider and CaveGenerators, to build the world. This allows for a modular and extensible world generation pipeline where different stages are handled by dedicated objects.

The ChunkGenerator acts as the bridge between the abstract definition of a world (zones, biomes, noise functions) and its concrete representation as blocks and entities within a chunk.

### Lifecycle & Ownership
- **Creation:** The ChunkGenerator is a heavyweight object. It is instantiated once per world during the server's bootstrap sequence. The constructor initializes the internal thread pool and all associated caches, making it an expensive operation that should not be performed frequently.
- **Scope:** An instance of ChunkGenerator persists for the entire lifetime of a game world. Its caches grow over time as more of the world is explored and generated.
- **Destruction:** The class implements a `shutdown` method which **must** be called when the world is unloaded. This method gracefully terminates the internal thread pool, allowing pending generation tasks to complete before stopping the threads. Failure to call `shutdown` will result in a resource leak, preventing the JVM from exiting cleanly.

## Internal State & Concurrency
- **State:** The ChunkGenerator is highly stateful and mutable. Its primary state is contained within its various cache objects (generatorCache, caveGeneratorCache, uniquePrefabCache, prefabLoadingCache). These caches store the results of expensive computations, forming a persistent memory of previously generated world data.

- **Thread Safety:** This class is designed to be thread-safe from the perspective of an external caller.
    - Public methods like `generate` are safe to call from any thread. They delegate the actual work to an internal, managed thread pool.
    - Internal state, particularly the caches, is managed by concurrent data structures or appropriate synchronization mechanisms to handle simultaneous access from multiple generation worker threads.
    - The use of `ThreadLocal` for ChunkGeneratorResource is the primary mechanism for achieving high parallelism without lock contention. Each worker thread operates on its own set of resources, minimizing shared mutable state.

    **WARNING:** While the public API is thread-safe, direct manipulation of the internal state (e.g., modifying a retrieved cache entry) is not supported and will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generate(seed, index, x, z, stillNeeded) | CompletableFuture | Complex | Asynchronously generates a full chunk at the given coordinates. This is the primary entry point for world generation. |
| shutdown() | void | O(N) | Initiates a graceful shutdown of the internal thread pool. N is the number of queued tasks. |
| getHeight(seed, x, z) | int | O(1) amortized | Retrieves the surface Y-coordinate for a given column. Heavily relies on the internal cache. |
| getCave(caveType, seed, x, z) | Cave | O(1) amortized | Retrieves cave data for a specific column. Accesses the CaveGeneratorCache. |
| getSpawnPoints(seed) | Transform[] | Complex | Calculates and returns all potential player spawn points for the given world seed. |
| getZoneBiomeResultAt(seed, x, z) | ZoneBiomeResult | O(1) amortized | Retrieves the Zone and Biome data for a specific column, a foundational piece of intermediate data. |

## Integration Patterns

### Standard Usage
The ChunkGenerator should be treated as a long-lived service retrieved from a central context, such as the World object. The primary interaction is to request chunk generation and process the result asynchronously.

```java
// Correctly retrieve the generator from the world context
ChunkGenerator generator = world.getChunkGenerator();

// Request a chunk at x=10, z=20 for world seed 12345
int seed = 12345;
long chunkIndex = ...; // Calculate chunk index
int chunkX = 10;
int chunkZ = 20;

CompletableFuture<GeneratedChunk> futureChunk = generator.generate(seed, chunkIndex, chunkX, chunkZ, null);

// Process the result asynchronously without blocking the main thread
futureChunk.thenAccept(generatedChunk -> {
    if (generatedChunk != null) {
        // Logic to process the fully generated chunk
        world.assimilateChunk(generatedChunk);
    }
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ChunkGenerator()`. This creates a new, expensive thread pool and an empty set of caches, defeating its purpose as a shared service and leading to severe performance degradation and resource contention.
- **Blocking on Futures:** Calling `future.join()` or `future.get()` on the CompletableFuture returned by `generate` from a time-critical thread (like the main server tick loop) is a severe anti-pattern. This will freeze the thread until generation is complete, causing server lag.
- **Forgetting Shutdown:** Failing to call `generator.shutdown()` when a world is unloaded will leak threads and memory. The application may not terminate correctly.
- **Concurrent Modification:** Do not attempt to modify the contents of data structures returned by getter methods (e.g., `getTimings`). These are intended for inspection only.

## Data Pipeline
The flow of data for a single chunk generation request is a multi-stage process orchestrated within the ChunkGenerator's thread pool.

> Flow:
> `generate(seed, x, z)` call -> Task submitted to **ChunkThreadPoolExecutor** -> Worker thread begins execution -> **ChunkGeneratorExecution** is created -> Caches are checked for Zone/Biome data -> If cache miss, `generateZoneBiomeResultAt` is called -> Noise functions are sampled -> Caches are checked for Heightmap data -> If cache miss, `generateHeight` is called -> Terrain is built block-by-block -> `generateCave` is called to carve tunnels -> Prefabs and structures are placed -> Entities are populated -> **GeneratedChunk** object is finalized -> `CompletableFuture` is completed -> Result is passed to `thenAccept` callback

