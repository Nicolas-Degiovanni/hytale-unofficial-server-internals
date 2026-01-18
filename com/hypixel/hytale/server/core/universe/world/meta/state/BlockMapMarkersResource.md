---
description: Architectural reference for BlockMapMarkersResource
---

# BlockMapMarkersResource

**Package:** com.hypixel.hytale.server.core.universe.world.meta.state
**Type:** Data Resource

## Definition
```java
// Signature
public class BlockMapMarkersResource implements Resource<ChunkStore> {
```

## Architecture & Concepts
The BlockMapMarkersResource class is a specialized data container, not a service. Its primary function is to encapsulate the state of all player-defined map markers that exist within the boundaries of a single world chunk. By implementing the generic Resource interface, it integrates into the server's chunk data management system, allowing marker data to be treated as a modular component of a chunk's overall state.

The core of this class is a `fastutil` Long2ObjectMap, which provides a memory and performance-optimized mapping between a packed block coordinate and its corresponding marker data. The conversion from a Vector3i position to a primitive long key is handled by the BlockUtil utility, a critical performance optimization that avoids object overhead for map keys.

Architecturally, the most significant feature is the static CODEC field. This object defines the contract for serialization and deserialization, enabling the Hytale engine to persist this resource to disk as part of the world save process and reconstruct it when a chunk is loaded. The CODEC contains custom logic to translate the internal long keys into String keys for storage, likely to ensure compatibility with human-readable or more portable data formats like JSON.

## Lifecycle & Ownership
- **Creation:** An instance of BlockMapMarkersResource is created under two main conditions:
    1.  By the Hytale serialization framework via its CODEC when a chunk is loaded from a ChunkStore and contains existing marker data.
    2.  Programmatically by the server when a player adds the *first* map marker to a chunk that previously had none.

- **Scope:** The lifetime of a BlockMapMarkersResource instance is strictly tied to the lifetime of its parent chunk in server memory. It exists only as long as the chunk is loaded and active.

- **Destruction:** The object is marked for garbage collection when its parent chunk is unloaded from the server. Prior to unloading, the server's persistence layer uses the CODEC to serialize the resource's state into the ChunkStore.

## Internal State & Concurrency
- **State:** The internal state is **mutable**. The `markers` map is directly modified by public API methods like addMarker and removeMarker. This class is a direct, in-memory representation of persisted world data.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be owned and operated on by a single thread, typically the main server thread that manages world ticks and chunk operations. Any concurrent modification from other threads without external synchronization will lead to data corruption and server instability.

## API Surface
The public API provides direct mechanisms for querying and manipulating the set of markers within a chunk.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceType() | static ResourceType | O(1) | Retrieves the unique, static identifier for this resource type from the BlockModule. |
| getMarkers() | Long2ObjectMap | O(1) | Returns a direct reference to the internal map of markers. **Warning:** The returned map is mutable and not a defensive copy. |
| addMarker(position, name, icon) | void | O(1) amortized | Adds a new marker or updates an existing one at the specified block position. |
| removeMarker(position) | void | O(1) amortized | Removes the marker at the specified block position. |
| clone() | Resource | O(N) | Creates a deep copy of the resource, where N is the number of markers. This is essential for safe state duplication. |

## Integration Patterns

### Standard Usage
This resource should always be accessed through the chunk that owns it. A game system must first acquire a reference to the loaded chunk and then request the resource by its type.

```java
// Example of a game system adding a marker to a chunk
Chunk targetChunk = world.getChunkAt(chunkPosition);

// Retrieve or create the resource for this chunk
BlockMapMarkersResource markers = targetChunk.getOrCreateResource(BlockMapMarkersResource.getResourceType());

// Mutate the resource state
Vector3i markerPosition = new Vector3i(10, 64, 20);
markers.addMarker(markerPosition, "Ancient Ruins", "ui_icon_tower");

// Signal to the chunk that its data has changed and needs to be saved
targetChunk.markDirty();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockMapMarkersResource()`. This creates an orphaned object that is not tracked by the chunk or the persistence engine. All instances must be managed by the parent chunk's resource system.
- **Cross-Thread Modification:** Never call addMarker or removeMarker from an asynchronous task or worker thread without synchronizing with the main server thread. This will cause a ConcurrentModificationException or silent data corruption.
- **Retaining Stale References:** Do not hold a reference to a BlockMapMarkersResource instance after its parent chunk may have been unloaded. This constitutes a memory leak and can lead to operating on invalid, out-of-date information.

## Data Pipeline
The BlockMapMarkersResource is a critical component in the data flow between in-memory game state and persistent storage.

**On Chunk Load (Deserialization):**
> Flow:
> ChunkStore on Disk -> Raw Serialized Bytes -> **CODEC Deserializer** -> **BlockMapMarkersResource Instance** -> In-Memory Chunk Resource Map

**On Chunk Save (Serialization):**
> Flow:
> In-Memory Chunk Resource Map -> **BlockMapMarkersResource Instance** -> **CODEC Serializer** -> Raw Serialized Bytes -> ChunkStore on Disk

