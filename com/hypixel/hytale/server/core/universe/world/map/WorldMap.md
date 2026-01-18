---
description: Architectural reference for WorldMap
---

# WorldMap

**Package:** com.hypixel.hytale.server.core.universe.world.map
**Type:** Transient Data Model

## Definition
```java
// Signature
public class WorldMap implements NetworkSerializable<UpdateWorldMap> {
```

## Architecture & Concepts
The WorldMap class serves as a server-side aggregator and serializer for the data required by a client to render its world map user interface. It is fundamentally a data-holding object, designed to be constructed, populated with map information, and then converted into a single, efficient network packet for transmission.

This class acts as a bridge between the server's internal world state and the client-facing network protocol. It collects two primary types of data:
1.  **Map Images:** Graphical representations of world chunks, stored in a performance-oriented `Long2ObjectMap` mapping a chunk index to a MapImage.
2.  **Points of Interest:** Named markers for specific locations, such as dungeons, cities, or player-set waypoints, stored in a `Map`.

Its core responsibility is to transform this collection of disparate data points into the `UpdateWorldMap` packet, which is the canonical representation of map data within the Hytale network protocol.

## Lifecycle & Ownership
-   **Creation:** A WorldMap instance is created on-demand by higher-level server logic, typically within a world or player session manager. It is not a persistent service but a temporary object for a specific data transmission task. The constructor requires an initial capacity hint for the number of chunks, optimizing memory allocation.
-   **Scope:** The object's lifetime is intentionally short. It exists only for the duration of the map data aggregation and serialization process.
-   **Destruction:** Once the `toPacket` method has been called and the resulting network packet has been passed to the network layer, the WorldMap instance has fulfilled its purpose. It holds no external resources and is subsequently garbage collected.

## Internal State & Concurrency
-   **State:** The internal state of WorldMap is highly mutable. Methods like `addPointOfInterest` directly modify the internal collections. A critical feature is the `packet` field, which acts as a write-once cache. After the first call to `toPacket`, the resulting packet is stored, and subsequent calls will return this cached instance without re-processing the internal maps.

    **Warning:** Any mutations to the WorldMap object after the first invocation of `toPacket` will be ignored and will not be reflected in the network data.

-   **Thread Safety:** This class is **not thread-safe**. The internal collections (`Object2ObjectOpenHashMap`, `Long2ObjectOpenHashMap`) are unsynchronized. Concurrent calls to `addPointOfInterest` or modifications to the underlying maps from multiple threads will lead to data corruption and unpredictable behavior. All access and modification must be externally synchronized, typically by confining its use to a single world-processing thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addPointOfInterest(id, name, type, pos) | void | O(1) | Adds a new map marker. Throws IllegalArgumentException if the marker ID already exists. |
| toPacket() | UpdateWorldMap | O(N) first call, O(1) subsequent | Serializes the internal state into a network packet. The first call iterates all N chunks. Subsequent calls return a cached packet. |
| getPointsOfInterest() | Map | O(1) | Returns a direct reference to the internal map of markers. **Warning:** Modifying this map directly is not recommended. |
| getChunks() | Long2ObjectMap | O(1) | Returns a direct reference to the internal map of chunk images. |

## Integration Patterns

### Standard Usage
The intended use involves instantiating the class, populating it with data from the server's world model, and then serializing it to a packet for network dispatch.

```java
// How a developer should normally use this
// 1. Create a new WorldMap instance with an expected size
WorldMap mapData = new WorldMap(player.getVisibleChunkCount());

// 2. Populate with points of interest from the world state
for (POI poi : world.getPointsOfInterest()) {
    mapData.addPointOfInterest(poi.getId(), poi.getName(), poi.getType(), poi.getTransform());
}

// 3. Populate with map image data (logic not shown in this class)
Long2ObjectMap<MapImage> chunks = mapData.getChunks();
// ... logic to fill the chunks map ...

// 4. Serialize to a packet and send it
UpdateWorldMap packet = mapData.toPacket();
player.getConnection().send(packet);
```

### Anti-Patterns (Do NOT do this)
-   **Modification after Serialization:** The most common error is modifying the WorldMap object after `toPacket` has been called. Due to internal caching, these changes will not be part of the serialized data. A new WorldMap instance must be created for new data.

    ```java
    // BAD: The second point of interest will not be in the packet
    WorldMap mapData = new WorldMap(10);
    mapData.addPointOfInterest("id1", "First POI", "icon", position1);
    UpdateWorldMap packet = mapData.toPacket(); // Packet is created and cached here
    mapData.addPointOfInterest("id2", "Second POI", "icon", position2); // This change is ignored
    player.getConnection().send(packet);
    ```

-   **Concurrent Access:** Do not share a WorldMap instance across multiple threads without explicit, external locking. The internal state is not protected against race conditions.

## Data Pipeline
The WorldMap class is a key component in the server-to-client data flow for rendering the in-game map.

> Flow:
> Server World State -> **WorldMap (Aggregation)** -> toPacket() (Serialization) -> UpdateWorldMap Packet -> Network Layer -> Client Map UI

