---
description: Architectural reference for CameraShakeConfig
---

# CameraShakeConfig

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Object

## Definition
```java
// Signature
public class CameraShakeConfig {
```

## Architecture & Concepts
The CameraShakeConfig class is a **Protocol Data Unit (PDU)**, a specialized Data Transfer Object designed for high-performance network communication. Its primary role is to encapsulate all parameters required to define a camera shake effect, enabling the server to command client-side rendering behavior. It is not a service or a manager; it is a pure data container with a strictly defined binary layout.

The class is a fundamental component of the client-server rendering protocol. It employs a custom, highly-optimized serialization format to minimize network overhead. This format is divided into two main parts:

1.  **Fixed-Size Block:** A 28-byte header containing primitive types (floats, booleans) and a bitmask for nullable fields. This allows for extremely fast, predictable reads of the most common data.
2.  **Variable-Size Block:** A subsequent data region for complex, nullable child objects like EasingConfig, OffsetNoise, and RotationNoise. The fixed-size block contains offsets pointing to the start of each of these objects within the variable block.

This hybrid structure provides the performance of a fixed layout while retaining the flexibility to omit optional, complex data, thereby saving bandwidth. The `nullBits` field is a critical optimization, acting as a bitmask to signal the presence or absence of nullable child objects without requiring extra bytes for null terminators.

## Lifecycle & Ownership
-   **Creation:** An instance of CameraShakeConfig is created under two distinct circumstances:
    1.  **Server-Side:** Instantiated directly via its constructor (`new CameraShakeConfig(...)`) by game logic that needs to trigger a camera shake effect on one or more clients.
    2.  **Client-Side:** Instantiated exclusively by the static `deserialize` method. This occurs deep within the network protocol layer when an incoming packet corresponding to this effect is being decoded.

-   **Scope:** The object is ephemeral and has a very short lifecycle. On the server, it exists only for the duration of the serialization process before being written to a network buffer. On the client, it exists from the moment of deserialization until it is consumed by the rendering engine, after which it is discarded. It is not intended to be stored or cached long-term.

-   **Destruction:** Ownership is transient. The object is managed by the Java Garbage Collector and becomes eligible for collection as soon as all references to it are dropped. This typically happens immediately after serialization on the server or after processing on the client.

## Internal State & Concurrency
-   **State:** The object is fully mutable, with all fields exposed publicly. This design choice prioritizes performance by avoiding the overhead of getters and setters, which is acceptable for an internal, short-lived data container. The state represents a complete, self-contained definition of a single camera shake effect.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context or a carefully managed, thread-isolated pipeline. For example, a Netty network thread may deserialize the object, but it must then be passed to the main game thread via a concurrent queue for safe processing. Modifying an instance from one thread while it is being read or serialized by another will lead to data corruption and undefined behavior.

## API Surface
The public contract is dominated by static methods for serialization, deserialization, and validation, which operate directly on Netty ByteBufs.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | CameraShakeConfig | O(N) | **[Client]** Constructs an object by reading from a binary buffer at a given offset. |
| serialize(buf) | void | O(N) | **[Server]** Writes the object's state into a binary buffer according to the defined protocol format. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size of a serialized object within a buffer without full deserialization. |
| validateStructure(buffer, offset) | ValidationResult | O(N) | Verifies that the data in a buffer represents a valid, non-corrupt object structure. Critical for security. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will consume when serialized. |

*N = size of the serialized data in bytes.*

## Integration Patterns

### Standard Usage
The class is used by the protocol layer to transfer data. The server creates and serializes it; the client receives the bytes and deserializes them.

```java
// SERVER: Creating and serializing the effect
CameraShakeConfig shake = new CameraShakeConfig(1.5f, 0.0f, false, easeIn, easeOut, offset, null);
Packet packet = new SomeClientEffectPacket(shake);
channel.writeAndFlush(packet); // This will internally call shake.serialize(buffer)

// CLIENT: Deserializing from a network buffer in a packet handler
// (Assumes 'buffer' is a ByteBuf containing the packet data)
CameraShakeConfig receivedShake = CameraShakeConfig.deserialize(buffer, buffer.readerIndex());
// The object is now ready to be passed to the rendering system
game.getEffectManager().applyCameraShake(receivedShake);
```

### Anti-Patterns (Do NOT do this)
-   **Client-Side Instantiation:** Do not use `new CameraShakeConfig()` on the client to process network data. The only valid way to create this object from a network stream is via the static `deserialize` method.
-   **State Mutation After Deserialization:** Once an object is deserialized on the client, it should be treated as immutable. Modifying its state can lead to unpredictable rendering behavior and desynchronization with server intent.
-   **Reusing Instances:** Do not reuse CameraShakeConfig instances for multiple effects. They are cheap to create and should be instantiated per-effect to ensure state integrity.
-   **Ignoring Validation:** Bypassing `validateStructure` on untrusted input can expose the client to malformed packets, potentially leading to buffer overflows, incorrect memory access, and crashes.

## Data Pipeline
The flow of CameraShakeConfig data is unidirectional from server to client.

> **Server Flow:**
> Game Logic -> `new CameraShakeConfig(...)` -> `serialize(ByteBuf)` -> Network Layer -> TCP/IP Stack

> **Client Flow:**
> TCP/IP Stack -> Network Layer -> Protocol Decoder -> `CameraShakeConfig.deserialize(ByteBuf)` -> Game Event Bus -> Rendering Engine

