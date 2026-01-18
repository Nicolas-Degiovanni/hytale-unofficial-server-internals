---
description: Architectural reference for IWorldGen
---

# IWorldGen

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface IWorldGen {
    // Methods defined below
}
```

## Architecture & Concepts
The IWorldGen interface defines the core contract for all server-side world generation systems. It serves as a high-level abstraction layer that decouples the world management and chunk loading systems from the specific, complex algorithms used to generate terrain, place biomes, and spawn structures.

Its primary responsibility is to asynchronously transform a set of world coordinates and a seed into a fully populated chunk of game data, represented by the GeneratedChunk object. By returning a CompletableFuture, the interface mandates that all generation work must be performed off the main server thread, preventing catastrophic server stalls and ensuring a responsive gameplay experience even under heavy load.

Implementations of this interface are expected to be highly complex, encapsulating noise functions, biome maps, structure placement logic, and other procedural generation algorithms. This interface is the entry point for the server's universe simulation to request new, unexplored sections of the world.

### Lifecycle & Ownership
As an interface, IWorldGen itself has no lifecycle. The following describes the lifecycle of a typical *implementation* of this contract.

-   **Creation:** An IWorldGen implementation is instantiated by the server's World or Universe manager during the initialization of a specific game world. It is tightly bound to the seed and configuration of that world.
-   **Scope:** The instance persists for the entire lifetime of the world it is responsible for generating. It is a long-lived, session-scoped service.
-   **Destruction:** The shutdown method is invoked when the world is unloaded or the server is shutting down. This allows the implementation to release any held resources, such as thread pools or file handles.

## Internal State & Concurrency
-   **State:** Implementations are inherently stateful and mutable. They maintain complex internal state derived from the world seed, including configured noise generators, biome placement rules, and potentially caches for frequently accessed procedural data. This state is not intended to be modified after initial creation.
-   **Thread Safety:** Implementations **must be thread-safe**. The generate method will be called concurrently from multiple worker threads as the server requests chunks for different players and systems simultaneously. The use of CompletableFuture signals a design where the heavy computational work is delegated to a dedicated thread pool managed by the implementation. All internal data structures must be designed to handle concurrent access without corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTimings() | WorldGenTimingsCollector | O(1) | Retrieves a collector for performance profiling data. May return null if profiling is disabled. |
| generate(int, long, int, int, LongPredicate) | CompletableFuture<GeneratedChunk> | O(N) Async | Asynchronously generates a chunk at the given coordinates. N is the volume of the chunk. This is the primary method of the interface. |
| getSpawnPoints(int) | Transform[] | O(1) | **Deprecated.** Retrieves a predefined set of potential spawn points. Use getDefaultSpawnProvider instead. |
| getDefaultSpawnProvider(int) | ISpawnProvider | O(1) | Returns a default spawn provider logic, typically used to find a safe surface location for new players. |
| shutdown() | void | O(1) | Signals the generator to clean up its internal resources, such as shutting down its thread pool. |

## Integration Patterns

### Standard Usage
The IWorldGen service is not meant to be used directly by most game logic. It is managed by the World system, which requests chunks as needed to expand the playable area.

```java
// Within a World management class
IWorldGen worldGenerator = this.getWorldGenerator();
CompletableFuture<GeneratedChunk> futureChunk = worldGenerator.generate(chunkX, chunkY, chunkZ, ...);

// Process the result asynchronously without blocking the main thread
futureChunk.thenAccept(generatedChunk -> {
    // Logic to integrate the new chunk into the live world state
    this.loadChunkData(generatedChunk);
});
```

### Anti-Patterns (Do NOT do this)
-   **Blocking on the Future:** Never call get() on the returned CompletableFuture from the main server thread. Doing so will freeze the entire server until the chunk is fully generated, leading to severe performance degradation and lag.
-   **Reusing Generators:** An IWorldGen instance is configured for a single world and its unique seed. Do not attempt to share or reuse a generator instance across different worlds.
-   **Ignoring the Shutdown Method:** Failing to call shutdown on world unload will lead to resource leaks, particularly leaving worker threads running unnecessarily.

## Data Pipeline
The IWorldGen interface is a critical stage in the world data pipeline, acting as the source of all procedural terrain data on the server.

> Flow:
> World Manager Chunk Request -> **IWorldGen.generate()** -> [Internal Thread Pool & Algorithms] -> CompletableFuture<GeneratedChunk> -> World Manager Chunk Integration -> Network Serialization -> Client Receives Chunk

