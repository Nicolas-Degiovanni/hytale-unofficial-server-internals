---
description: Architectural reference for WorldSavingStatus
---

# WorldSavingStatus

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class WorldSavingStatus implements Packet {
```

## Architecture & Concepts
The WorldSavingStatus packet is a simple, fixed-size Data Transfer Object (DTO) used within the Hytale network protocol. Its sole purpose is to communicate a binary state change from the server to the client, specifically indicating whether a world-saving operation is currently in progress.

This class is a fundamental component of the client-server status synchronization mechanism. It is not a service or manager but rather a piece of data in transit. The static fields, such as PACKET_ID and FIXED_BLOCK_SIZE, serve as metadata for the protocol's central packet registry and deserialization engine. This metadata allows the network layer to efficiently identify, validate, and decode the incoming byte stream into a concrete WorldSavingStatus object without complex reflection or dynamic lookups.

It represents a server-authoritative notification, enabling the client's user interface to provide feedback to the player, such as displaying a "Saving..." icon or temporarily disabling actions that could conflict with a save operation.

## Lifecycle & Ownership
- **Creation:**
    - **Sending Peer (Server):** Instantiated directly when the server's world persistence logic begins or ends a save operation. For example: `new WorldSavingStatus(true)`.
    - **Receiving Peer (Client):** Instantiated by the static `deserialize` factory method, which is invoked by the network protocol decoder when a byte stream with PACKET_ID 233 is received.
- **Scope:** Extremely short-lived. An instance exists only for the duration of its serialization and transmission, or from its deserialization until it is processed by a packet handler. It is immediately eligible for garbage collection afterward.
- **Destruction:** Managed by the Java Garbage Collector. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Mutable. The primary state is the public boolean field isWorldSaving. While technically mutable, instances of this packet should be treated as immutable after creation or deserialization. Its state represents a snapshot of the server's status at a specific point in time.
- **Thread Safety:** **Not thread-safe.** This object is designed to be created, serialized, and processed within a single thread context, typically a Netty I/O thread or a main game thread. It must not be shared or modified across multiple threads without external synchronization, which would be a significant anti-pattern.

## API Surface
The public API is designed for interaction with the network protocol engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (233). |
| serialize(ByteBuf) | void | O(1) | Encodes the object's state into the provided Netty byte buffer. |
| computeSize() | int | O(1) | Returns the constant size of the packet payload in bytes (1). |
| deserialize(ByteBuf, int) | WorldSavingStatus | O(1) | **Static Factory.** Decodes a new instance from a byte buffer at a given offset. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static.** Verifies if the buffer contains enough data to decode the packet. |

## Integration Patterns

### Standard Usage
On the client, a packet handler receives the deserialized object and updates the relevant UI system. The handler should not hold a reference to the packet.

```java
// Hypothetical client-side packet handler
public class InterfacePacketHandler {
    private final UIManager uiManager;

    public void handle(WorldSavingStatus statusPacket) {
        // The packet's data is extracted and immediately used.
        // The packet object itself is then discarded.
        boolean isSaving = statusPacket.isWorldSaving;
        uiManager.showWorldSavingIndicator(isSaving);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not modify the isWorldSaving field after the packet has been received. It is a status report, not a mutable state container for client-side logic.
- **Long-Term Storage:** Do not store instances of this packet in caches or long-lived collections. Its purpose is immediate processing. Storing it can lead to stale data.
- **Cross-Thread Sharing:** Never pass an instance of this packet to another thread for processing without ensuring a proper happens-before relationship. It is not designed for concurrent access.

## Data Pipeline
The flow for this packet is unidirectional from the server to the client.

> Flow:
> Server World Service -> **WorldSavingStatus (Created)** -> Protocol Encoder -> Netty Channel -> Network -> Client Protocol Decoder -> **WorldSavingStatus (Reconstructed)** -> Client Packet Handler -> UI Manager Update

