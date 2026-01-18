---
description: Architectural reference for RequestServerAccess
---

# RequestServerAccess

**Package:** com.hypixel.hytale.protocol.packets.serveraccess
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class RequestServerAccess implements Packet {
```

## Architecture & Concepts
The **RequestServerAccess** class is a specialized Data Transfer Object (DTO) that represents a network packet. It operates within the Hytale Protocol Layer and is a critical component of the initial server connection handshake.

Its primary function is to communicate a client's intent to establish a server session with a specific visibility level (e.g., Private, Friends, Public). It also carries the client's externally-facing network port, a key piece of information for network address translation (NAT) traversal and peer-to-peer connectivity.

This packet is not a service or a manager; it is a simple, inert data container. Its structure is rigidly defined by static constants like **PACKET_ID** and **FIXED_BLOCK_SIZE**, ensuring consistent serialization and deserialization between the client and server. The design prioritizes performance and low allocation overhead, evident in the static **deserialize** and **validateStructure** methods that operate directly on Netty's **ByteBuf**.

## Lifecycle & Ownership
- **Creation:** An instance is created by the client-side connection logic immediately before attempting to join or host a game server. The constructor is called with the desired **Access** level and the discovered external port.
- **Scope:** The object's lifetime is extremely short. On the client, it exists only long enough to be serialized into a **ByteBuf** for network transmission. On the server, it is instantiated during deserialization, processed by a single packet handler, and then becomes eligible for garbage collection. It does not persist beyond the scope of a single network event.
- **Destruction:** The Java Garbage Collector reclaims the object's memory once it is no longer referenced, which occurs almost immediately after its use in the network pipeline.

## Internal State & Concurrency
- **State:** The internal state is mutable and consists of two public fields: **access** and **externalPort**. The class is a Plain Old Java Object (POJO) and does not encapsulate or hide its data. It performs no caching.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and processed within a single thread, typically a Netty event loop thread. Sharing an instance across multiple threads without external synchronization will lead to race conditions and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (250). |
| serialize(ByteBuf buf) | void | O(1) | Encodes the packet's state into the provided Netty byte buffer. |
| deserialize(ByteBuf buf, int offset) | RequestServerAccess | O(1) | Static factory method. Decodes a new instance from a byte buffer at a given offset. |
| validateStructure(ByteBuf buffer, int offset) | ValidationResult | O(1) | Static method. Performs a low-level check to ensure the buffer contains enough data to form a valid packet. |

## Integration Patterns

### Standard Usage
This packet is intended to be used exclusively by the core networking framework. A client's connection manager instantiates it, and the protocol pipeline handles its serialization and transmission.

```java
// Client-side logic initiating a server connection
import com.hypixel.hytale.protocol.packets.serveraccess.Access;

// Assume 'networkClient' is an existing network layer abstraction
short discoveredPort = networkClient.getUpnpPort();
RequestServerAccess packet = new RequestServerAccess(Access.Private, discoveredPort);
networkClient.sendPacket(packet);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and resend the same packet instance. These objects are cheap to create and should be treated as immutable after their initial construction.
- **Manual Serialization:** Avoid calling **serialize** or **deserialize** directly. These methods are low-level and intended for use only by the protocol's encoding and decoding pipeline handlers.
- **Cross-Thread Sharing:** Never pass an instance of this packet to another thread for modification. If data needs to be shared, extract the primitive values (**access**, **externalPort**) and pass those instead.

## Data Pipeline
The flow of this data object is linear and unidirectional through the network stack.

> **Client Flow:**
> Connection Logic -> **new RequestServerAccess()** -> Network Pipeline -> Packet Encoder (calls serialize) -> TCP/UDP Socket

> **Server Flow:**
> TCP/UDP Socket -> Packet Decoder (calls deserialize) -> **RequestServerAccess instance** -> Packet Handler -> Server Access Controller -> Discarded

