---
description: Architectural reference for ChunkWorldMap
---

# ChunkWorldMap

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap.provider.chunk
**Type:** Singleton

## Definition
```java
// Signature
public class ChunkWorldMap implements IWorldMap {
```

## Architecture & Concepts
The ChunkWorldMap class is a concrete implementation of the IWorldMap strategy interface. It is responsible for generating world map imagery on the server by rendering individual world chunks and composing them into a single map object.

This class functions as a high-level orchestrator, not a direct renderer. Its primary architectural role is to manage the asynchronous, parallel generation of map tiles. For a given set of chunk coordinates, it dispatches concurrent rendering tasks to the ImageBuilder utility. This design is critical for server performance, preventing the main game loop from blocking on a potentially long-running, CPU-intensive, or I/O-bound operation.

By leveraging CompletableFuture, it provides a non-blocking API contract, allowing the calling system to chain subsequent logic (like serialization and network transmission) to be executed upon the completion of the map generation.

### Lifecycle & Ownership
- **Creation:** Eagerly instantiated as a static final field upon class loading. This is a compile-time singleton that exists as soon as the ChunkWorldMap class is referenced by the JVM.
- **Scope:** Application-scoped. A single instance, accessed via ChunkWorldMap.INSTANCE, persists for the entire lifetime of the server process.
- **Destruction:** The instance is not explicitly destroyed. It is eligible for garbage collection only when the server's class loader is unloaded, which typically occurs during a full server shutdown.

## Internal State & Concurrency
- **State:** This class is entirely stateless. It maintains no instance fields and does not cache results between invocations. All required context, such as the World object and the list of chunks to render, is provided as method arguments.
- **Thread Safety:** The class is inherently thread-safe. Due to its stateless design, the public methods can be invoked concurrently from multiple threads without any risk of data corruption or race conditions. Each call to the generate method operates on a new, isolated set of CompletableFuture objects.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWorldMapSettings() | WorldMapSettings | O(1) | Returns a static, hardcoded configuration object for map display settings. |
| generate(world, width, height, chunks) | CompletableFuture<WorldMap> | O(N) | Asynchronously orchestrates the rendering of N chunks. Returns a future that completes with the composite WorldMap. |
| generatePointsOfInterest(world) | CompletableFuture<Map> | O(1) | Stub implementation. Immediately returns a completed future with an empty map of markers. |

## Integration Patterns

### Standard Usage
This component is a singleton and should be accessed directly via its static INSTANCE field. The primary use case is to request a map generation and process the result asynchronously.

```java
// Standard asynchronous invocation
LongSet chunksToRender = ...;
World myWorld = ...;

CompletableFuture<WorldMap> mapFuture = ChunkWorldMap.INSTANCE.generate(myWorld, 256, 256, chunksToRender);

mapFuture.thenAccept(worldMap -> {
    // This block executes on a worker thread when the map is ready.
    // Send the completed worldMap to the client or cache it.
    System.out.println("Map generation complete for " + worldMap.getChunks().size() + " chunks.");
});
```

### Anti-Patterns (Do NOT do this)
- **Redundant Instantiation:** Do not use `new ChunkWorldMap()`. The class is designed as a singleton, and creating new instances is unnecessary and defeats the design pattern. Always use `ChunkWorldMap.INSTANCE`.
- **Blocking Server Threads:** Calling `mapFuture.get()` on a critical server thread (e.g., the main game tick loop) is a severe anti-pattern. This will block the thread until the entire map generation is complete, potentially causing server-wide freezes or crashes. Always use asynchronous callbacks like `thenApply` or `thenAccept`.

## Data Pipeline
The ChunkWorldMap acts as a fan-out/fan-in orchestrator in the server's data pipeline for map generation.

> Flow:
> World Map Service Request -> **ChunkWorldMap.generate()** -> Dispatches N parallel tasks to ImageBuilder -> ImageBuilder renders chunk data from world storage -> Futures complete with image data -> **ChunkWorldMap** aggregates images into a WorldMap object -> World Map Service -> Network Serialization -> Client
---

