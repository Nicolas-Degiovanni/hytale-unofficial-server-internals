---
description: Architectural reference for UpdateWorldMap
---

# UpdateWorldMap

**Package:** com.hypixel.hytale.protocol.packets.worldmap
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class UpdateWorldMap implements Packet {
```

## Architecture & Concepts
The UpdateWorldMap class is a Data Transfer Object (DTO) that represents a single, concrete message within the Hytale network protocol. Its sole purpose is to encapsulate data for updating the client's view of the world map. This packet is sent from the server to the client to transmit new map chunk data, add new map markers, or remove existing ones.

As an implementation of the Packet interface, it is a fundamental component of the network layer. It provides the logic for serializing its state into a Netty ByteBuf for network transmission and deserializing a ByteBuf back into a structured object upon receipt. The binary layout is highly optimized, using a bitmask for nullable fields and offsets to variable-length data blocks to minimize payload size. This class is not intended to hold long-term state; it is a message that is processed and immediately discarded.

## Lifecycle & Ownership
- **Creation:**
    - **Sending Peer (Server):** Instantiated via its constructor (`new UpdateWorldMap(...)`) by the server's map management system when changes need to be broadcast to a client.
    - **Receiving Peer (Client):** Instantiated by the network protocol layer when an incoming byte stream is identified with Packet ID 241. The static `deserialize` method is invoked by a packet factory or dispatcher, which constructs the object from the raw buffer.

- **Scope:** The object's lifetime is extremely short and confined to the scope of a single network tick or event processing block. It exists only to be passed from the deserializer to the appropriate packet handler.

- **Destruction:** It becomes eligible for garbage collection immediately after its contents have been processed by a packet handler and the relevant data has been transferred to a more permanent state manager, such as a client-side WorldMap service.

## Internal State & Concurrency
- **State:** UpdateWorldMap is a mutable container for map data. Its public fields—chunks, addedMarkers, and removedMarkers—can be directly accessed and modified after instantiation. However, it should be treated as immutable after being deserialized or before being passed to the serializer.

- **Thread Safety:** This class is **not thread-safe**. It is designed for synchronous processing within a single network thread (e.g., a Netty event loop). All serialization, deserialization, and handling must occur on the same thread to prevent data corruption and race conditions. Any handoff to another thread, such as the main game thread, requires external synchronization or a thread-safe queuing mechanism.

## API Surface
The primary contract is defined by the Packet interface and the static utility methods for buffer manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static UpdateWorldMap | O(N) | Constructs an UpdateWorldMap object by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. |
| computeSize() | int | O(N) | Calculates the total byte size the packet will occupy when serialized. Used for pre-allocating buffers. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check of the buffer to ensure it contains a valid packet structure without full deserialization. |
| getId() | int | O(1) | Returns the static network identifier for this packet type, which is 241. |

*Complexity O(N) is relative to the total number of elements in the chunks, markers, and other arrays.*

## Integration Patterns

### Standard Usage
The class is exclusively used by the network protocol engine. A packet handler receives the deserialized object, extracts its data, and forwards it to the relevant game systems for processing.

```java
// Executed on a network thread (e.g., Netty Event Loop)
void handlePacket(UpdateWorldMap packet) {
    // Get the client's primary world map state manager
    ClientWorldMapManager mapManager = Game.getInstance().getWorldMapManager();

    // Apply the changes from the packet to the game state
    if (packet.chunks != null) {
        mapManager.addOrUpdateChunks(packet.chunks);
    }
    if (packet.addedMarkers != null) {
        mapManager.addMarkers(packet.addedMarkers);
    }
    if (packet.removedMarkers != null) {
        mapManager.removeMarkersById(packet.removedMarkers);
    }
    
    // The 'packet' object is now out of scope and will be garbage collected.
}
```

### Anti-Patterns (Do NOT do this)
- **State Retention:** Do not hold references to UpdateWorldMap objects after they have been processed. Their data should be copied into persistent game state models immediately.

- **Cross-Thread Access:** Never access a packet object from a different thread than the one that deserialized it without explicit, careful synchronization. This will lead to undefined behavior.

- **Manual Instantiation on Client:** The client-side code should never create instances using `new UpdateWorldMap()`. These packets are strictly messages received from the server.

- **Ignoring Validation:** Bypassing `validateStructure` in security-sensitive contexts can expose the server or client to denial-of-service attacks via maliciously crafted packets.

## Data Pipeline
UpdateWorldMap acts as a data carrier in the server-to-client information flow for the world map system.

> Flow:
> Server Map System -> **UpdateWorldMap (Serialization)** -> Network Layer (TCP) -> Client Network Layer -> **UpdateWorldMap (Deserialization)** -> Client Packet Handler -> Client WorldMapManager -> UI Render Thread

