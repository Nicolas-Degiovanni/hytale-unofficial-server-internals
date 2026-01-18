---
description: Architectural reference for UVMotion
---

# UVMotion

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class UVMotion {
```

## Architecture & Concepts

The UVMotion class is a passive data structure, or Data Transfer Object, that serves as a fundamental component of the Hytale network protocol. It exclusively defines the properties of a dynamic texture coordinate (UV) animation, which is likely consumed by the rendering engine for particle effects, animated block textures, or shader effects.

Architecturally, this class is not a service or manager; it is a data contract. Its primary role is to be a portable, language-agnostic representation of a visual effect that can be efficiently serialized for network transmission and deserialized by a client.

The binary format is highly optimized for performance and low bandwidth usage. It employs a fixed-size block for predictable, non-nullable fields, followed by a variable-size block for optional data. The presence of optional fields is controlled by a leading `nullBits` bitmask, a common pattern in high-performance network protocols to avoid transmitting unnecessary data.

## Lifecycle & Ownership

-   **Creation:** An instance of UVMotion is created under two primary circumstances:
    1.  **Deserialization:** The static `deserialize` factory method instantiates the object when decoding an incoming network packet on a client or server. This is the most common creation path during gameplay.
    2.  **Direct Instantiation:** Game logic, such as a content creation tool or a server-side system defining a new particle effect, will create an instance via its constructor before passing it to the network layer for serialization.

-   **Scope:** The object's lifetime is typically ephemeral and context-bound. Once deserialized, it exists only long enough for the relevant game system (e.g., a ParticleRenderer) to consume its data. It is not intended to be a long-lived object.

-   **Destruction:** UVMotion instances are managed by the Java Garbage Collector. There are no manual resource management or `close` methods. Ownership is transient and is not transferred between systems; systems read its state and then discard the reference.

## Internal State & Concurrency

-   **State:** The class maintains a mutable state. All fields are public, allowing for direct, low-overhead modification. This design choice prioritizes performance over encapsulation, which is appropriate for an internal, short-lived DTO. It does not cache any data; it is a direct representation of its fields.

-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives.

    **WARNING:** Accessing and modifying a UVMotion instance from multiple threads without external synchronization will lead to data races and undefined behavior. It is designed to be created, processed, and discarded within the confines of a single thread, such as a Netty I/O worker or the main game update thread.

## API Surface

The public contract is dominated by static methods for serialization and validation, reflecting its role as a network protocol building block.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static UVMotion | O(N) | **Primary Factory.** Deserializes data from a buffer into a new UVMotion instance. N is the length of the texture string. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided buffer according to the defined binary protocol. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will consume when serialized. Useful for pre-allocating buffers. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | **Security Critical.** Scans the buffer to validate lengths and structure without full deserialization. Essential for preventing denial-of-service attacks via malformed packets. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the size of an encoded UVMotion directly from a buffer. Allows a network stream parser to skip over the object. |

## Integration Patterns

### Standard Usage

The primary interaction pattern involves deserializing the object from a network buffer and passing the resulting data to a consumer system.

```java
// In a network packet handler, where 'buffer' is the incoming ByteBuf
// and 'readOffset' is the current position in the buffer.

ValidationResult result = UVMotion.validateStructure(buffer, readOffset);
if (!result.isOk()) {
    throw new ProtocolException("Invalid UVMotion data: " + result.getReason());
}

UVMotion motion = UVMotion.deserialize(buffer, readOffset);
int bytesConsumed = UVMotion.computeBytesConsumed(buffer, readOffset);
// Advance buffer reader index by bytesConsumed...

// Pass the immutable data to the rendering system
renderingSystem.applyTextureMotion(motion);
```

### Anti-Patterns (Do NOT do this)

-   **State Reuse:** Do not attempt to reuse a UVMotion instance by populating it with new data. The `deserialize` method always returns a new object, and this pattern should be followed. Modifying a shared instance can lead to unpredictable visual artifacts.
-   **Ignoring Validation:** Never call `deserialize` on data from an untrusted network source without first calling `validateStructure`. A malicious client could send a packet with an invalid string length, causing the server to attempt a massive memory allocation, leading to an OutOfMemoryError.
-   **Cross-Thread Modification:** Do not pass a UVMotion instance to another thread for processing while retaining a reference and modifying it on the original thread. This is a classic data race. If data must be shared, use a copy constructor or create a new immutable representation.

## Data Pipeline

UVMotion acts as a data payload that flows from a definition source, across the network, and to a rendering consumer.

> **Serialization Flow (Server -> Client):**
> Game Logic -> `new UVMotion(...)` -> **UVMotion.serialize()** -> Network Packet Buffer -> TCP/UDP Socket

> **Deserialization Flow (Client):**
> TCP/UDP Socket -> Netty I/O Thread -> Network Packet Buffer -> **UVMotion.deserialize()** -> Game Logic (e.g., Particle System) -> Render Command

