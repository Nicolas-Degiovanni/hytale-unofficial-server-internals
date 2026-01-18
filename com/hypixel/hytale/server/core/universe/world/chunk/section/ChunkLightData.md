---
description: Architectural reference for ChunkLightData
---

# ChunkLightData

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.section
**Type:** Immutable Data Structure

## Definition
```java
// Signature
public class ChunkLightData {
```

## Architecture & Concepts

ChunkLightData is a highly optimized, memory-efficient data structure that encapsulates all lighting information for a 32x32x32 volume of blocks, typically a chunk section. It is not a simple 3D array; instead, it represents lighting data as a **sparse octree** stored in a compact, serialized format within a Netty ByteBuf.

This architectural choice is critical for performance and memory management. A naive 3D array for lighting would consume a fixed 64KB per section (32*32*32 blocks * 2 bytes/block). The octree structure provides significant compression by allowing large, uniformly-lit volumes (e.g., solid rock underground, open sky) to be represented by a single node in the tree.

Each block's light value is a 16-bit short, composed of four 4-bit channels:
*   **Red Channel:** Light from red-emitting sources.
*   **Green Channel:** Light from green-emitting sources.
*   **Blue Channel:** Light from blue-emitting sources.
*   **Sky Channel:** Light from the sky.

The core of the class is the static `getTraverse` method, which performs a recursive descent through the in-memory octree to resolve a light value for a specific block coordinate. This traversal is logarithmic in complexity, making lookups extremely fast.

## Lifecycle & Ownership

-   **Creation:** ChunkLightData instances are not meant to be constructed directly. They are created through two primary mechanisms:
    1.  **Deserialization:** Instantiated by the chunk loading pipeline via the static `deserialize` method when a chunk is read from disk or received over the network.
    2.  **Building:** Constructed on the server by the `ChunkLightDataBuilder` during the light propagation calculation phase.

-   **Scope:** The lifecycle of a ChunkLightData object is tightly bound to its parent `ChunkSection`. It persists in memory for as long as the chunk section is loaded and active in the world.

-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection once its parent `ChunkSection` is unloaded and all references to it are released. The underlying Netty ByteBuf is reference-counted and will be released back to its pool upon garbage collection.

## Internal State & Concurrency

-   **State:** This class is **effectively immutable**. Its core state, the `light` ByteBuf and the `changeId`, are set only during construction and are never modified. All public methods are read-only queries.

-   **Thread Safety:** Due to its immutable design, ChunkLightData is **inherently thread-safe for read operations**. Multiple engine threads (e.g., Rendering, AI, Physics) can safely and concurrently query lighting values from the same instance without requiring any locks or synchronization primitives. This is a crucial performance characteristic for the engine.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChangeId() | short | O(1) | Returns the version identifier for this light data, used for caching and dirty-checking. |
| getSkyLight(x, y, z) | byte | O(log N) | Retrieves the 4-bit sky light value at the given local coordinates. |
| getBlockLight(index) | short | O(log N) | Retrieves the packed 16-bit RGB block light value for a given block index. |
| getLightRaw(index) | short | O(log N) | The primary lookup method. Retrieves the raw, packed 16-bit value for all four light channels. |
| serialize(ByteBuf) | void | O(N) | Traverses the entire internal octree and writes its compressed form to the provided buffer for network or disk persistence. |
| deserialize(ByteBuf, int) | ChunkLightData | O(N) | A static factory method that parses a serialized octree from a buffer and constructs a new ChunkLightData instance. |

## Integration Patterns

### Standard Usage

A system, such as the rendering engine, should retrieve the ChunkLightData instance from a `ChunkSection` and query it for the light values needed for vertex processing.

```java
// Hypothetical example from a meshing or rendering system
ChunkSection activeSection = world.getChunkSectionAt(position);
ChunkLightData lightData = activeSection.getLightData();

// Query the light at a specific block within the section
byte skyLight = lightData.getSkyLight(localX, localY, localZ);
short blockLight = lightData.getBlockLight(localX, localY, localZ);

// Use light values to shade a vertex
vertex.setLight(skyLight, blockLight);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new ChunkLightData()`. The internal ByteBuf has a highly specific binary format representing the octree. Invalid construction will lead to `IndexOutOfBoundsException` or engine crashes. Always use the `ChunkLightDataBuilder` on the server or rely on the `deserialize` method.
-   **Long-Term Caching:** Do not hold references to ChunkLightData instances beyond the scope of their parent `ChunkSection`. When a chunk is unloaded, its light data should be garbage collected. Caching stale instances can lead to memory leaks and incorrect lighting.
-   **Modification via Reflection:** Attempting to access and modify the internal `light` ByteBuf is a severe violation of the class's contract. Its immutability is fundamental to engine stability and thread safety.

## Data Pipeline

The flow of light data from creation on the server to consumption on the client follows a clear serialization and deserialization path.

> **Server-Side Generation:**
> Light Propagation Engine → ChunkLightDataBuilder → **ChunkLightData** (In-Memory Octree) → `serialize()` → Network Packet

> **Client-Side Consumption:**
> Network Packet → `deserialize()` → **ChunkLightData** (In-Memory Octree) → Voxel Meshing Engine → Rendered Frame

