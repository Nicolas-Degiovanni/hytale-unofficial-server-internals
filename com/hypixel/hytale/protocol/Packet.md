---
description: Architectural reference for the Packet interface, the core contract for all network communication.
---

# Packet

**Package:** com.hypixel.hytale.protocol
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface Packet {
   int getId();

   void serialize(@Nonnull ByteBuf var1);

   int computeSize();
}
```

## Architecture & Concepts
The Packet interface is the fundamental contract for all data transfer objects (DTOs) within the Hytale network protocol. It establishes a standardized structure for messages sent between the client and server, forming the bedrock of the entire network communication layer.

This interface does not represent a concrete message but rather defines the essential capabilities any network message must possess: a unique identifier, a method for serialization into a raw byte stream, and a way to pre-calculate its size for efficient buffer allocation.

It serves as the critical abstraction layer between high-level game logic systems and the low-level network transport, which is managed by the Netty framework. Game systems create and populate objects that implement Packet, while the network pipeline consumes these objects, using the interface methods to handle the mechanics of network transmission without needing to know the specifics of each message type.

### Lifecycle & Ownership
The lifecycle described here applies to **concrete implementations** of the Packet interface, not the interface itself.

-   **Creation:** Packet objects are instantiated on-demand.
    -   **Outbound:** Game logic systems (e.g., a PlayerInputSystem) create a new Packet instance when an event occurs that must be communicated to the remote peer.
    -   **Inbound:** A central PacketRegistry or factory, part of the network decoding pipeline, instantiates a specific Packet implementation after reading its unique ID from the incoming byte stream.
-   **Scope:** Transient. Packet objects are extremely short-lived and designed for single use. They exist only to carry data for a single serialization or deserialization operation.
-   **Destruction:** Packet instances are eligible for garbage collection immediately after being processed. For outbound packets, this is after the `serialize` method is called by the network encoder. For inbound packets, this is after they have been passed to and processed by the relevant game logic handler.

## Internal State & Concurrency
-   **State:** Concrete implementations of Packet are inherently stateful and mutable. They are simple data containers whose fields are populated at creation time. They do not typically contain complex logic.
-   **Thread Safety:** **CRITICAL WARNING:** Packet implementations are **not thread-safe** and must never be shared across threads. They are designed to be created, populated, and passed to the network pipeline from a single thread, typically the main game thread or a dedicated world thread. The Netty I/O threads handle the actual serialization and deserialization in a thread-safe manner within the pipeline, but the Packet objects themselves must not be mutated concurrently.

## API Surface
The public contract is minimal, focusing exclusively on the requirements of the network pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static, unique integer ID for this packet type. Used for registration and decoding. |
| serialize(ByteBuf) | void | O(N) | Writes the packet's internal state into the provided Netty ByteBuf. N is the number of fields. |
| computeSize() | int | O(1) to O(N) | Calculates the total size in bytes that this packet will occupy after serialization. Essential for pre-allocating buffers. Complexity depends on field types (e.g., variable-length strings). |

## Integration Patterns

### Standard Usage
A system creates a specific packet implementation, populates its fields, and hands it off to a network service for transmission. The developer should never call `serialize` directly.

```java
// Example of sending a hypothetical player position update
PlayerPositionPacket positionPacket = new PlayerPositionPacket(player.getX(), player.getY(), player.getZ());

// The NetworkManager is responsible for writing the packet to the channel,
// which triggers the internal serialization pipeline.
networkManager.sendPacket(positionPacket);
```

### Anti-Patterns (Do NOT do this)
-   **Packet Reuse:** Do not attempt to cache and reuse Packet objects. They are lightweight and designed to be single-use. Reusing a packet can lead to sending stale data or unpredictable serialization errors. Always create a new instance for each message.
-   **Holding References:** Do not maintain long-term references to Packet objects after they have been sent or received. They should be considered fire-and-forget.
-   **Manual Serialization:** Avoid calling the `serialize` method directly in game logic. This is the sole responsibility of the network pipeline's encoders. Bypassing the pipeline can corrupt the byte stream.

## Data Pipeline
The Packet interface is the central data structure in the network data flow.

> **Outbound Flow (Sending a Packet):**
> Game Logic -> `new ConcretePacket(data)` -> Network Service -> Netty Channel Pipeline -> Encoder (calls `packet.serialize()`) -> TCP/IP Stack

> **Inbound Flow (Receiving a Packet):**
> TCP/IP Stack -> Netty Channel Pipeline -> Decoder (reads ID, creates `new ConcretePacket()`, populates fields) -> Packet Handler -> Game Logic Update

