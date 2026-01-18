---
description: Architectural reference for GeneratorChunkWorldMap
---

# GeneratorChunkWorldMap

**Package:** com.hypixel.hytale.server.worldgen.map
**Type:** Service / Adapter

## Definition
```java
// Signature
public class GeneratorChunkWorldMap extends ChunkWorldMap {
```

## Architecture & Concepts
The GeneratorChunkWorldMap class is a concrete implementation of the ChunkWorldMap abstraction. Its primary architectural role is to serve as an **adapter** between the procedural world generation system and the server's world map service. It translates low-level configuration data from a ChunkGenerator into high-level data structures required by clients to render and interact with the world map.

This class is responsible for two distinct functions:
1.  **Point of Interest (POI) Generation:** It asynchronously scans the world generator's configuration for unique, map-visible prefabs (e.g., dungeons, villages) and transforms them into MapMarker objects.
2.  **Map Settings Compilation:** It aggregates biome and zone data from the generator to construct the fundamental WorldMapSettings, which includes biome-to-color mappings and default map scaling rules.

By encapsulating this logic, GeneratorChunkWorldMap decouples the world map system from the specific implementation details of any given world generator, adhering to the Liskov Substitution Principle for its parent ChunkWorldMap. The mandatory injection of an Executor service underscores its design for non-blocking, asynchronous operation, preventing world initialization from stalling the main server thread.

### Lifecycle & Ownership
-   **Creation:** An instance of GeneratorChunkWorldMap is created during the server's world initialization phase. It is not instantiated directly but is likely constructed by a factory or dependency injection container that provides its required dependencies: a fully configured ChunkGenerator and a shared Executor service.
-   **Scope:** The object's lifetime is tightly coupled to the World instance it describes. It persists as long as the corresponding world is loaded in memory. It is not a global singleton; a distinct instance would exist for each procedurally generated world.
-   **Destruction:** The object is eligible for garbage collection when the World it is associated with is unloaded from the server. There are no explicit cleanup or shutdown methods.

## Internal State & Concurrency
-   **State:** This class is effectively **stateless**. Its internal fields, ChunkGenerator and Executor, are final and injected at construction. It does not cache data or maintain any mutable state between method invocations. Each call to its public methods computes a result fresh from the underlying ChunkGenerator.

-   **Thread Safety:** The class is designed to be **thread-safe**. The `generatePointsOfInterest` method is explicitly asynchronous, offloading its work to the provided Executor to avoid blocking the caller. The `getWorldMapSettings` method performs read-only operations on the ChunkGenerator. Thread safety is therefore contingent on the immutability of the ChunkGenerator's configuration post-initialization, which is a safe assumption in the context of the server's world loading sequence.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generatePointsOfInterest(World world) | CompletableFuture | O(N) | Asynchronously generates a map of unique prefab locations. N is the number of unique prefabs defined in the generator. The future completes exceptionally if an error occurs during processing. |
| getWorldMapSettings() | WorldMapSettings | O(M) | Synchronously builds the complete map settings object. M is the total number of biomes across all zones. Throws IllegalArgumentException if duplicate or invalid biome IDs are detected. |

## Integration Patterns

### Standard Usage
This class is intended to be used by a higher-level world management service during the setup of a new world. The service retrieves the map data and forwards it to clients, typically upon their first connection to that world.

```java
// Within a world initialization service
ChunkGenerator generator = world.getChunkGenerator();
Executor backgroundExecutor = server.getAsyncExecutor();

// Instantiate the map provider
ChunkWorldMap mapProvider = new GeneratorChunkWorldMap(generator, backgroundExecutor);

// Asynchronously fetch POIs and send to clients when complete
mapProvider.generatePointsOfInterest(world).thenAccept(pois -> {
    // Network logic to send POIs to players in the world
});

// Synchronously get settings and send to clients immediately
WorldMapSettings settings = mapProvider.getWorldMapSettings();
// Network logic to send settings to players
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation without Context:** Do not use `new GeneratorChunkWorldMap(..)` with a partially configured generator. The class expects the ChunkGenerator to be fully initialized with all its zones and biomes.
-   **Blocking on the Main Thread:** Calling `generatePointsOfInterest(world).get()` on the main server thread is a critical performance anti-pattern. It negates the benefit of the asynchronous design and will cause the server to hang while POIs are being processed.
-   **Frequent Re-computation:** The data generated by this class is static for a given world seed. Do not call `getWorldMapSettings` repeatedly; its result should be computed once and cached by the calling service for the lifetime of the world.

## Data Pipeline
The class functions as a transformation stage in the server's data pipeline for initializing the client-side world map.

> **Point of Interest Flow:**
> ChunkGenerator Config -> UniquePrefabEntry[] -> **GeneratorChunkWorldMap** -> CompletableFuture<Map<String, MapMarker>> -> Network Packet Encoder -> Client UI

> **Settings Flow:**
> ChunkGenerator Config -> Zone[] / Biome[] -> **GeneratorChunkWorldMap** -> WorldMapSettings -> Network Packet Encoder -> Client UI

