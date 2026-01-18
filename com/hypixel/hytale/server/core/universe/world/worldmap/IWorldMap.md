---
description: Architectural reference for IWorldMap
---

# IWorldMap

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IWorldMap {
   WorldMapSettings getWorldMapSettings();

   CompletableFuture<WorldMap> generate(World var1, int var2, int var3, LongSet var4);

   CompletableFuture<Map<String, MapMarker>> generatePointsOfInterest(World var1);

   default void shutdown() {
   }
}
```

## Architecture & Concepts
The IWorldMap interface defines a formal contract for services responsible for generating and retrieving world map data on the server. It acts as a critical abstraction layer, decoupling the core world simulation from the specifics of map rendering and data retrieval. This allows for multiple map generation strategies—such as on-the-fly rendering, pre-computed cache lookups, or a hybrid approach—without altering the consuming services.

The most significant architectural decision is the mandated use of CompletableFuture for all data generation operations. This establishes a non-blocking, asynchronous pattern for map generation, which is essential for server performance. Generating map tiles or querying for points of interest can be computationally expensive or I/O-bound; performing these tasks on the main server thread would lead to catastrophic stalls in the game loop (tick lag). By returning a future, the IWorldMap implementation can offload the work to a background thread pool, notifying the caller only upon completion.

The interface also logically separates the generation of topographical map data (generate) from semantic location data (generatePointsOfInterest). This separation of concerns allows the two systems to evolve independently and source their data from different places. For example, the map may be generated from live chunk data, while points of interest are loaded from a static, pre-defined world database.

## Lifecycle & Ownership
- **Creation:** Concrete implementations of IWorldMap are instantiated by a world-scoped service locator or dependency injection container during the initialization of a World instance. The specific implementation class is likely determined by server configuration.
- **Scope:** An IWorldMap instance is tightly coupled to a single World instance. Its lifecycle is expected to match the lifecycle of the world it represents, persisting as long as the world is loaded in memory.
- **Destruction:** The default shutdown method is invoked by the parent World when it is being unloaded or during a full server shutdown. Implementations must use this hook to release any managed resources, such as thread pools, file handles, or database connections.

## Internal State & Concurrency
- **State:** As an interface, IWorldMap is stateless. However, any concrete implementation is expected to be highly stateful, managing caches for map tiles, region data, and points of interest to avoid redundant computation.
- **Thread Safety:** This contract is designed for a concurrent environment. Implementations **must be thread-safe**. Calls to its methods will originate from the main server thread, but the work is executed on background threads. All internal mutable state, especially caches, must be protected using appropriate synchronization primitives like locks, concurrent collections, or atomic operations to prevent data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWorldMapSettings() | WorldMapSettings | O(1) | Retrieves the static configuration object that governs map generation parameters. |
| generate(World, int, int, LongSet) | CompletableFuture<WorldMap> | O(N) | Asynchronously generates the topographical map for a specified set of regions. N is proportional to the size of the LongSet. |
| generatePointsOfInterest(World) | CompletableFuture<Map<String, MapMarker>> | O(P) | Asynchronously queries and returns all known points of interest for the entire world. P is the number of POIs. |
| shutdown() | void | O(1) | Releases all underlying resources held by the implementation. Failure to call this will result in resource leaks. |

## Integration Patterns

### Standard Usage
The intended use is to request map data asynchronously and chain subsequent logic to the returned CompletableFuture, ensuring the main server thread is never blocked.

```java
// A World-scoped service retrieves its IWorldMap instance
IWorldMap mapSystem = world.getService(IWorldMap.class);

// Request POIs without blocking the game loop
mapSystem.generatePointsOfInterest(world)
    .thenAcceptAsync(points -> {
        // This code executes on a background thread upon completion
        // Process the points and perhaps send them to a player
        player.getConnection().sendPacket(new PacketUpdatePOIs(points));
    }, world.getExecutor());
```

### Anti-Patterns (Do NOT do this)
- **Blocking on the Main Thread:** Never call get() or join() on the returned CompletableFuture from the main server thread. This defeats the purpose of the asynchronous design and will freeze the server.
- **Ignoring Shutdown:** Implementations may manage their own thread pools or hold open file handles. Forgetting to call shutdown() when a world is unloaded is a guaranteed resource leak.
- **State Leakage:** Returning mutable objects from an implementation's cache without defensive copying can lead to severe concurrency issues if multiple threads modify the object simultaneously.

## Data Pipeline
The IWorldMap interface sits between world management services and the underlying world data sources, orchestrating an asynchronous data flow.

> Flow:
> Player Map Request -> World Service -> **IWorldMap.generate()** -> [Offloaded to Worker Thread Pool] -> Read Chunk Data / Query Database -> Generate Map Tiles & Markers -> CompletableFuture Completion -> Network Encoder -> TCP Packet to Client

