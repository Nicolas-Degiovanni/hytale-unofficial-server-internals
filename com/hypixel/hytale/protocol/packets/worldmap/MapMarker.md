---
description: Architectural reference for MapMarker
---

# MapMarker

**Package:** com.hypixel.hytale.protocol.packets.worldmap
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class MapMarker {
```

## Architecture & Concepts
The MapMarker class is a data structure that represents a single point of interest on the in-game world map. It is not a managed game entity but rather a transient Data Transfer Object (DTO) designed exclusively for network serialization and deserialization.

Its primary architectural role is to define the precise binary layout for map marker information transmitted between the server and client. The serialization format is highly optimized for network efficiency, employing a hybrid fixed-block and variable-block layout.

- **Fixed-Block:** The first 54 bytes of a serialized MapMarker contain a fixed-layout structure. This includes a one-byte bitmask (nullBits) to indicate the presence of nullable fields, followed by the inline data for fixed-size fields (like Transform) and offsets for variable-size fields.
- **Variable-Block:** Following the fixed block, a variable-length data region contains the actual content for strings and arrays (id, name, contextMenuItems). The offsets in the fixed-block point to the start of each data element within this region.

This design allows for extremely fast parsing, as the location and presence of every field can be determined without scanning the entire data structure.

## Lifecycle & Ownership
- **Creation:** A MapMarker instance is created under two circumstances:
    1. **Serialization (Outbound):** The game server's logic instantiates a MapMarker and populates its fields to represent a marker that needs to be sent to a client.
    2. **Deserialization (Inbound):** The static method MapMarker.deserialize is invoked by a higher-level network packet handler, which constructs a MapMarker instance from a raw Netty ByteBuf.

- **Scope:** The object's lifetime is intentionally brief. It exists only for the duration of a single network operation or a single game-tick processing cycle. It is a snapshot of data, not a persistent state container.

- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as it is no longer referenced by the network stack or game logic. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The state of a MapMarker is fully mutable. All data fields are public and can be modified directly after instantiation. This design prioritizes performance and ease of use for the constructing/serializing and deserializing/consuming logic paths.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. Concurrent access, particularly writing to fields from one thread while another thread is calling serialize or reading from them, will result in undefined behavior, data corruption, or serialization failures.

**WARNING:** A MapMarker instance must only be accessed and modified by a single thread at any given time. It is typically confined to the Netty network thread during deserialization or the main game thread during creation and processing.

## API Surface
The public contract is dominated by static methods for handling the binary protocol.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | MapMarker | O(N) | **[Primary Entry Point]** Constructs a MapMarker object by reading from a ByteBuf at a given offset. N is the size of the variable data. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. N is the size of the variable data. Throws ProtocolException if data exceeds limits. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a security and integrity check on the binary data in a buffer without fully deserializing it. Crucial for rejecting malformed packets early. |
| computeSize() | int | O(N) | Calculates the total number of bytes this object will consume when serialized. Useful for pre-allocating buffers. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the size of a serialized MapMarker directly from a buffer. Used by parent structures to advance the buffer reader index. |

## Integration Patterns

### Standard Usage
A MapMarker is never used in isolation. It is always processed by a higher-level packet handler that manages a collection of markers. The standard pattern involves calling the static deserialize method to populate an object from the network stream.

```java
// A network handler receives a buffer containing map data
// For each marker in the payload, it deserializes it.
int currentOffset = ...;
MapMarker marker = MapMarker.deserialize(byteBuf, currentOffset);

// The handler then passes the DTO to a game system
worldMapManager.addOrUpdateMarker(marker);

// The buffer offset is advanced for the next object
currentOffset += MapMarker.computeBytesConsumed(byteBuf, currentOffset);
```

### Anti-Patterns (Do NOT do this)
- **State Persistence:** Do not hold references to MapMarker objects across multiple frames or game ticks. They represent a point-in-time snapshot and should be converted into a more stable internal game state representation immediately.
- **Cross-Thread Modification:** Do not deserialize a MapMarker on a network thread and then modify its public fields from the main game thread without proper synchronization. This will lead to severe race conditions. The object should be treated as immutable after being passed between threads.
- **Ignoring Validation:** Do not process incoming data without first calling validateStructure on the buffer. Bypassing this step exposes the client or server to crashes and potential exploits from malformed network packets.

## Data Pipeline
The MapMarker class is a critical serialization/deserialization stage in the world map data pipeline.

> **Inbound Flow (Client):**
> Raw TCP Stream -> Netty ByteBuf -> Packet Decoder -> **MapMarker.deserialize** -> WorldMapManager -> UI Render

> **Outbound Flow (Server):**
> Game Event (e.g., Player creates waypoint) -> WorldMapManager -> **new MapMarker()** -> Packet Encoder -> **MapMarker.serialize** -> Netty ByteBuf -> Raw TCP Stream

