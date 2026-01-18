---
description: Architectural reference for ServerTags
---

# ServerTags

**Package:** com.hypixel.hytale.protocol.packets.setup
**Type:** Transient

## Definition
```java
// Signature
public class ServerTags implements Packet {
```

## Architecture & Concepts
The ServerTags class is a Data Transfer Object (DTO) representing a specific network message within the Hytale protocol, identified by PACKET_ID 34. Its sole purpose is to encapsulate a collection of server-specific key-value pairs, referred to as "tags". These tags are used to communicate server metadata, rules, or configuration from the server to the client during the connection setup phase.

This class is a fundamental component of the low-level protocol layer. It is not intended to contain any game logic. Instead, it provides a strict contract for serialization and deserialization, translating the in-memory map of tags into a well-defined binary format for network transmission and vice-versa. The implementation works directly with Netty's ByteBuf, signifying its position at the boundary between raw network I/O and the higher-level packet handling system.

The wire format defined by this class includes features for efficiency and safety, such as a nullable bitfield to avoid sending empty data, variable-length integers (VarInt) for compact size encoding, and explicit size validation to prevent buffer overflows and denial-of-service attacks.

### Lifecycle & Ownership
-   **Creation:**
    1.  **Inbound:** An instance is created by the protocol's deserialization pipeline when a network buffer with packet ID 34 is processed. The static factory method **deserialize** is the entry point for this process.
    2.  **Outbound:** An instance is manually instantiated by server-side logic when it needs to broadcast its configuration to a connecting client.

-   **Scope:** The lifetime of a ServerTags object is extremely brief and transactional.
    -   An inbound instance exists only for the duration of its processing by a single packet handler.
    -   An outbound instance exists only until it is passed to the network encoder and serialized into a ByteBuf.

-   **Destruction:** The object is managed by the Java garbage collector. Given its short-lived nature, it is typically eligible for collection almost immediately after use. There are no manual cleanup or resource release requirements.

## Internal State & Concurrency
-   **State:** The class holds a single, mutable state field: a `Map<String, Integer>` named tags. This field can be null, representing an absence of tags, which is handled explicitly by the serialization format. The object's state is not immutable and can be modified after creation, although this is not a recommended practice.

-   **Thread Safety:** **This class is not thread-safe.** The internal HashMap is not synchronized. It is designed to be created, populated, and processed within a single-threaded context, such as a Netty event loop thread.
    -   **WARNING:** Sharing a ServerTags instance across multiple threads without external synchronization will lead to race conditions and unpredictable behavior. Do not modify the tags map from one thread while another thread is serializing or reading from it.

## API Surface
The public API is dominated by static methods for protocol handling and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ServerTags | O(N) | Constructs a ServerTags instance by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the internal state of the instance into the provided ByteBuf. Throws ProtocolException if constraints are violated. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the instance without performing the write. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check of a ByteBuf to ensure it contains a structurally valid ServerTags packet. |
| getId() | int | O(1) | Returns the constant network identifier for this packet type (34). |

*N represents the total size of the data within the tags map.*

## Integration Patterns

### Standard Usage
The primary integration pattern is within a packet handling system. An inbound packet is deserialized and passed to a handler, which then uses the data to configure client-side systems.

```java
// Example of a packet handler processing an inbound ServerTags packet
public void handleServerTags(ServerTags packet) {
    if (packet.tags == null) {
        // No tags were sent by the server
        return;
    }

    // Apply server configuration based on received tags
    Integer maxPlayers = packet.tags.get("max_players");
    if (maxPlayers != null) {
        this.gameConfig.setMaxPlayers(maxPlayers);
    }

    Integer pvpMode = packet.tags.get("pvp_mode");
    if (pvpMode != null) {
        this.world.setPvpMode(PvpMode.fromId(pvpMode));
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not serialize the same ServerTags instance multiple times if its internal map is modified between writes. Packets should be treated as immutable once created.
-   **Cross-Thread Modification:** Do not create a ServerTags packet on one thread and pass it to a network I/O thread for serialization if other threads can still modify the underlying map. This is a severe race condition.
-   **Ignoring Size Limits:** Do not attempt to populate the tags map with more than 4,096,000 entries or with keys longer than 4,096,000 bytes. The **serialize** method will throw a ProtocolException, halting network communication.

## Data Pipeline
The ServerTags class is a data structure that exists at a specific point in the network data flow.

> **Inbound Flow:**
> Raw TCP Bytes -> Netty ByteBuf -> Hytale Packet Decoder -> **ServerTags Instance** -> Client-side Packet Handler -> Game State Configuration

> **Outbound Flow:**
> Server-side Game Logic -> **new ServerTags(map)** -> Hytale Packet Encoder -> Netty ByteBuf -> Raw TCP Bytes

