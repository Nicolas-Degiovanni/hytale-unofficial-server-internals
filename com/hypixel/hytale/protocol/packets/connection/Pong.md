---
description: Architectural reference for Pong
---

# Pong

**Package:** com.hypixel.hytale.protocol.packets.connection
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class Pong implements Packet {
```

## Architecture & Concepts
The Pong packet is a fundamental component of the low-level connection protocol, acting as the direct response to a Ping packet. Its primary purpose is to facilitate the calculation of network latency (Round-Trip Time or RTT) and to communicate basic connection health metrics between the client and server.

As a DTO, it is a pure data container with no associated logic beyond serialization and deserialization. Its structure is defined with a fixed size of 20 bytes, ensuring highly predictable and efficient processing by the network pipeline. This fixed-size nature is critical for performance in the initial packet decoding stages, where the system can allocate and read a known quantity of bytes without needing to parse variable-length fields.

The inclusion of a nullable InstantData field allows for precise timestamping, while the PongType enum suggests its use in different phases of the connection lifecycle, such as initial handshakes versus periodic keep-alive checks.

## Lifecycle & Ownership
- **Creation:** A Pong object is instantiated under two specific conditions:
    1.  **Inbound:** The static `deserialize` factory method is invoked by the network protocol decoder when a raw byte buffer with Packet ID 3 is received from the network.
    2.  **Outbound:** An instance is created directly via its constructor on the responding endpoint (e.g., a server responding to a client's Ping) immediately before serialization.

- **Scope:** Transient and extremely short-lived. A Pong object exists only for the duration of its processing within a single network event loop tick. It is not designed to be cached, stored, or referenced beyond the immediate scope of the handler that processes it.

- **Destruction:** The object becomes eligible for garbage collection as soon as the network event handler completes its work. For inbound packets, this is after the latency is calculated. For outbound packets, this is after its state has been written to the network buffer by the encoder.

## Internal State & Concurrency
- **State:** Mutable. The fields of a Pong object are public and can be modified after instantiation. However, by convention and design, it should be treated as an immutable value object once deserialized or populated for serialization.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, processed, and discarded exclusively within a single network thread (e.g., a Netty event loop). Sharing a Pong instance across multiple threads without explicit, external synchronization is a severe anti-pattern and will result in race conditions and undefined behavior.

## API Surface
The public contract is dominated by static methods for serialization, reflecting its role as a passive data structure manipulated by the network pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | Pong | O(1) | **Critical:** Static factory that constructs a Pong object from a raw network buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the internal state of the object into the provided network buffer for transmission. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a pre-check to ensure a buffer contains enough readable bytes to constitute a valid Pong packet. |
| clone() | Pong | O(1) | Creates a deep copy of the packet instance. |

## Integration Patterns

### Standard Usage
The Pong packet is almost never interacted with directly by high-level game logic. Its lifecycle is managed entirely by the low-level connection and protocol handling layers. A typical inbound flow involves the network layer deserializing the packet and passing it to a connection manager to update latency statistics.

```java
// Executed within a Netty channel handler or packet dispatcher
// The 'buffer' is an incoming ByteBuf from the network

ValidationResult result = Pong.validateStructure(buffer, offset);
if (result.isOk()) {
    Pong pongPacket = Pong.deserialize(buffer, offset);
    
    // Use the packet data to update connection state
    connection.updateLatency(pongPacket.id, pongPacket.time);
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not modify a Pong object after it has been deserialized. It represents a network-authoritative snapshot in time. Modifying its fields can corrupt latency calculations and connection state.
- **Long-Term Storage:** Do not store Pong instances in collections or as member variables. They are transient and should be processed immediately. If you need to retain information, extract the primitive values (id, time) and discard the packet object.
- **Manual Serialization:** Avoid calling the `serialize` method directly in application code. The network pipeline's encoder is responsible for invoking this method as part of its outbound message processing.

## Data Pipeline
The Pong packet is a response mechanism. Its data flow is simple and symmetrical, traveling from a network buffer into an in-memory object representation, and vice-versa.

> **Inbound Flow (e.g., Client receiving from Server):**
> Network ByteBuf -> Netty Channel Decoder -> **Pong.deserialize** -> ConnectionManager -> Latency Calculation

> **Outbound Flow (e.g., Server responding to Client):**
> Ping Received -> ConnectionManager -> **new Pong(...)** -> Netty Channel Encoder -> **Pong.serialize** -> Network ByteBuf

