---
description: Architectural reference for HytaleGenerator
---

# HytaleGenerator

**Package:** com.hypixel.hytale.builtin.hytalegenerator.plugin
**Type:** Singleton

## Definition
```java
// Signature
public class HytaleGenerator extends JavaPlugin {
```

## Architecture & Concepts
The HytaleGenerator class is the primary service provider for procedural world generation within the Hytale server. As a JavaPlugin, it is a modular, lifecycle-aware component loaded by the server core at startup. Its fundamental responsibility is to act as the bridge between the server's world management system and the complex, multi-stage chunk generation pipeline.

This class orchestrates the entire generation process. It receives abstract ChunkRequest objects, which specify coordinates, a seed, and a desired world structure. It then translates these requests into concrete generation tasks executed by a specialized ChunkGenerator.

Key architectural features include:

-   **Generator Caching:** It maintains a cache of ChunkGenerator instances, keyed by the ChunkRequest.GeneratorProfile. This avoids the expensive process of re-initializing the generation pipeline for every chunk when the profile (seed and world structure) is the same.
-   **Pipeline Factory:** Its most critical role is to construct the `NStagedChunkGenerator`. This involves reading configuration from `WorldStructureAsset` and `SettingsAsset`, dynamically building a sequence of generation stages (e.g., Biome, Terrain, Props), and configuring data buffers for communication between stages.
-   **Concurrency Management:** The class abstracts away the complexity of multi-threaded world generation. It creates and manages its own dedicated thread pools: a single-threaded `mainExecutor` for request submission and result handling, and a multi-threaded `concurrentExecutor` for performing the computationally intensive work in parallel.
-   **Asset Hot-Reloading:** It is designed to support live reloading of world generation assets. It registers a listener with the AssetManager to be notified of changes, at which point it safely tears down and rebuilds its internal state, including the generator cache and thread pools.

## Lifecycle & Ownership
-   **Creation:** An instance of HytaleGenerator is created by the server's plugin loading mechanism during server initialization. The `JavaPluginInit` object provided to the constructor contains the necessary context and registries from the server core.
-   **Scope:** The HytaleGenerator instance is a singleton within the plugin container and persists for the entire server session. Its lifecycle is tied directly to the server's lifecycle.
-   **Destruction:** The object is eligible for garbage collection when the server shuts down or the plugin is unloaded. The `concurrentExecutor` is explicitly shut down and awaited upon termination during the `reloadGenerators` or `loadExecutors` flow, ensuring a graceful shutdown of worker threads.

## Internal State & Concurrency
-   **State:** The internal state of HytaleGenerator is highly mutable and consists of cached generators, thread pools, and configuration values derived from assets.
    -   **generators:** A `Map` caching fully constructed `NStagedChunkGenerator` instances. This cache is volatile and is completely cleared when assets are reloaded.
    -   **executors:** The `mainExecutor` and `concurrentExecutor` encapsulate the threading model. Their configuration (e.g., thread count) is derived from `SettingsAsset` and can change during a hot-reload.
    -   **chunkGenerationSemaphore:** A `Semaphore` with a single permit, effectively acting as a mutex. This is the primary mechanism for ensuring thread safety.

-   **Thread Safety:** This class is thread-safe. Concurrency is managed through a combination of an asynchronous, future-based API and a strict internal locking model.
    -   All access to the mutable `generators` cache, both for reads (get) and writes (creation), is guarded by the `chunkGenerationSemaphore`.
    -   The `reloadGenerators` method acquires this same semaphore, creating a write lock that blocks all new generation tasks until the reload process is complete. This prevents race conditions where a chunk request might use a stale or partially reconfigured generator.
    -   The `submitChunkRequest` method is the designated entry point for concurrent calls from the server. It immediately offloads work to its internal `mainExecutor`, preventing the caller thread from blocking and ensuring that all requests are serialized through a single control point before generation begins.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| submitChunkRequest(ChunkRequest) | CompletableFuture | O(1) | Submits a request for chunk generation. The operation is asynchronous. The returned future completes with the GeneratedChunk or an exception. Submission itself is constant time. |
| createStagedChunkGenerator(...) | NStagedChunkGenerator | O(B + R) | Factory method to construct a new, fully configured staged generator pipeline. Complexity is proportional to the number of Biomes (B) and unique Runtimes (R) defined in the assets. |

## Integration Patterns

### Standard Usage
The server's world management system should treat this class as the authoritative source for world generation. It submits requests and asynchronously consumes the results.

```java
// Assume 'worldGenProvider' is a handle to the HytaleGenerator instance
ChunkRequest request = new ChunkRequest(...);

CompletableFuture<GeneratedChunk> futureChunk = worldGenProvider.submitChunkRequest(request);

futureChunk.thenAccept(generatedChunk -> {
    // Process the completed chunk on the main server thread
    world.applyChunkData(generatedChunk);
});
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new HytaleGenerator()`. The class must be instantiated by the server's plugin framework, which provides the required `JavaPluginInit` context.
-   **External State Modification:** Do not attempt to access or modify the internal `generators` cache or executors from outside this class. This will break the concurrency model and lead to unpredictable behavior.
-   **Synchronous Blocking:** Do not call `future.get()` on a `CompletableFuture` returned by `submitChunkRequest` from a critical server thread. This will block the thread and can cause severe server performance degradation. Use asynchronous callbacks like `thenAccept` or `handle`.

## Data Pipeline
The flow of data for a single chunk generation request is a highly orchestrated, asynchronous pipeline.

> Flow:
> ChunkRequest -> **HytaleGenerator.submitChunkRequest** -> mainExecutor -> **HytaleGenerator.getGenerator** (Cache Hit/Miss) -> NStagedChunkGenerator.generate -> concurrentExecutor (Parallel Stage Execution) -> GeneratedChunk -> CompletableFuture Completion -> Server World Thread

---

