---
description: Architectural reference for WorldMapManager
---

# WorldMapManager

**Package:** com.hypixel.hytale.server.core.universe.world.worldmap
**Type:** Per-World Singleton

## Definition
```java
// Signature
public class WorldMapManager extends TickingThread {
```

## Architecture & Concepts
The WorldMapManager is a stateful, thread-safe service responsible for the entire lifecycle of in-game map data for a single World instance. It acts as the central authority for generating, caching, and distributing map imagery and markers to players within that world.

Its architecture is designed around asynchronous, on-demand generation to prevent blocking the main server thread. It operates on its own dedicated thread, inherited from TickingThread, to handle expensive operations like image generation and cache eviction without impacting world simulation performance.

Key responsibilities include:
- **Image Generation:** Orchestrates the creation of map image tiles by delegating to a pluggable IWorldMap generator. This allows different map generation strategies to be used.
- **Asynchronous Caching:** Manages a multi-layered cache for map images. It maintains a map of completed images and a map of in-flight generation tasks (as CompletableFutures) to prevent redundant work.
- **Cache Eviction:** Periodically scans the image cache and unloads tiles that are no longer visible to any player, conserving memory. This is managed by a keep-alive mechanism.
- **Marker Management:** Aggregates and provides map markers from various sources through a MarkerProvider system. This includes dynamic markers like player locations and static markers like points of interest (POIs).
- **Client Synchronization:** While it does not directly send network packets, it provides the data required by each player's WorldMapTracker to synchronize map state with the client.

The WorldMapManager is a critical bridge between the persistent world data, the dynamic game state, and the player's user interface.

### Lifecycle & Ownership
- **Creation:** Instantiated by its parent World object during the world's initialization phase. A single WorldMapManager exists for each active World.
- **Scope:** The lifecycle of a WorldMapManager is strictly bound to its parent World. It persists as long as the World is loaded in memory.
- **Destruction:** The manager is shut down when its parent World is unloaded. The TickingThread is stopped, and the active IWorldMap generator is given a chance to clean up its resources via its shutdown method.

## Internal State & Concurrency
- **State:** The WorldMapManager is highly stateful and mutable. Its primary state consists of several concurrent collections:
    - **images:** A Long2ObjectConcurrentHashMap serving as the hot cache for generated MapImage tiles, keyed by a packed chunk index.
    - **generating:** A Long2ObjectConcurrentHashMap that tracks ongoing image generation tasks. Storing CompletableFuture instances here prevents duplicate generation requests for the same tile.
    - **markerProviders:** A ConcurrentHashMap mapping string identifiers to MarkerProvider implementations, allowing for extensible map marker logic.
    - **pointsOfInterest:** A ConcurrentHashMap caching pre-generated POI markers.

- **Thread Safety:** This class is designed to be thread-safe.
    - It extends TickingThread, isolating its core update loop (tick method) to a single, dedicated "WorldMap" thread.
    - Public methods intended for external access, such as getImageAsync, are safe to be called from any thread (typically the main world thread).
    - All shared mutable state is stored in thread-safe collections (e.g., ConcurrentHashMap) or managed via atomic primitives (e.g., AtomicInteger in ImageEntry). This ensures safe interaction between the manager's own thread and external callers.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setGenerator(IWorldMap) | void | O(N) | Configures the map generator. Passing null disables the world map. This is a destructive operation that clears all cached data. N is the number of players. |
| getImageAsync(long index) | CompletableFuture | O(1) | Asynchronously requests a map image tile. Returns a future that completes with the image. If the image is not cached, this triggers generation. This is the primary method for data retrieval. |
| getImageIfInMemory(long index) | MapImage | O(1) | Performs a non-blocking check for a cached image. Returns the image if present, otherwise null. Does not trigger generation. |
| addMarkerProvider(String, MarkerProvider) | void | O(1) | Registers a provider for a specific type of dynamic map marker. |
| clearImages() | void | O(C) | Immediately purges all cached and in-progress map images. C is the number of cached images. |
| unloadImages() | void | O(C) | Scans the image cache and removes any images that are no longer visible to players and whose keep-alive timer has expired. |

## Integration Patterns

### Standard Usage
The WorldMapManager is not typically used directly. Instead, systems like a player's WorldMapTracker interact with it to acquire map data needed for the client. The pattern is always asynchronous to avoid blocking.

```java
// A simplified example of how another system might use the manager
WorldMapManager manager = world.getWorldMapManager();
long imageIndex = ChunkUtil.indexChunk(x, z);

// Asynchronously request an image. The callback will handle sending it to the client.
manager.getImageAsync(imageIndex).thenAccept(mapImage -> {
    // Send the mapImage to the player over the network
    player.sendPacket(mapImage);
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new WorldMapManager()`. The World is solely responsible for its creation and lifecycle. Always retrieve it from the World instance.
- **Blocking on Futures:** Calling `get()` on the CompletableFuture returned by getImageAsync from a performance-critical thread (like the main server tick) is a severe anti-pattern. This will stall the thread and cause server lag. Always use asynchronous callbacks like `thenAccept`.
- **Frequent Generator Swapping:** Calling setGenerator repeatedly is expensive. It clears all caches and re-initializes state for all players. The generator should be set once during world setup.

## Data Pipeline
The flow of data for a single map tile request demonstrates the manager's role as an orchestrator and caching layer.

> Flow:
> Player Movement -> WorldMapTracker identifies required tile -> **WorldMapManager.getImageAsync()** -> Cache Miss -> **IWorldMap.generate()** -> World Chunk Data -> Processed into MapImage -> **WorldMapManager** caches result -> CompletableFuture completes -> WorldMapTracker receives MapImage -> Network Packet sent to Client

