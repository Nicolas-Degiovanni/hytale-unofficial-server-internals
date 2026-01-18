---
description: Architectural reference for the Particle data model.
---

# Particle

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Model

## Definition
```java
// Signature
public class Particle {
```

## Architecture & Concepts

The Particle class is a Data Transfer Object (DTO) that represents the complete definition of a particle effect within the Hytale network protocol. It is not a live, in-world particle entity, but rather the serialized blueprint used to create such entities.

Its primary role is to serve as the in-memory representation of a custom binary format sent over the network. The serialization protocol is highly optimized for performance, employing a hybrid structure:

1.  **Fixed-Size Block:** A 133-byte block at the beginning of the serialized data holds predictable, fixed-size fields like enums, floats, and booleans. This allows for extremely fast, direct-offset reads.
2.  **Offset Table:** Following the fixed block, an 8-byte section contains integer offsets pointing to the location of variable-sized data.
3.  **Variable-Size Block:** All variable-length data, such as the texturePath string and the animationFrames map, are appended at the end of the structure. This prevents a single large field from shifting the offsets of all subsequent fields.

A single byte, `nullBits`, acts as a bitmask at the very beginning of the payload. Each bit corresponds to a nullable field, indicating its presence or absence in the data stream. This is a critical optimization to minimize payload size when optional features of a particle are not used.

This class is a fundamental building block for the rendering and gameplay systems, acting as the bridge between raw network data and the client-side particle simulation engine.

### Lifecycle & Ownership

-   **Creation:** A Particle object is almost exclusively created by the network layer when a packet is decoded, via the static `deserialize` factory method. It can also be created programmatically by game logic (e.g., an effects editor or spawner) before being passed to the `serialize` method for network transmission. The copy constructor and `clone` method facilitate state duplication for modification without altering a shared template.
-   **Scope:** Transient and short-lived. The lifetime of a Particle instance is typically bound to the processing of a single network packet or a single game tick. It is treated as a value object and is not intended to be stored as a long-lived component.
-   **Destruction:** The object is managed by the Java Garbage Collector. It holds no native resources and requires no explicit cleanup. It becomes eligible for collection as soon as all references to it are dropped.

## Internal State & Concurrency

-   **State:** The Particle class is a mutable data container. All of its fields are public, allowing for direct, high-performance modification. This design choice prioritizes speed over encapsulation, which is common in performance-critical DTOs within game engines. The object's state is self-contained and does not reference or cache data from other systems.
-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. All fields are exposed for direct mutation, making concurrent access inherently unsafe. It is designed to be owned and operated on by a single thread at a time, such as a Netty I/O thread during deserialization or the main game thread during simulation logic.

## API Surface

The primary contract of this class revolves around its static serialization and validation methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static Particle | O(N) | Constructs a Particle object from a binary representation in a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the given ByteBuf according to the custom binary protocol. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs crucial safety checks on a buffer before deserialization. Verifies offsets, lengths, and bounds to prevent crashes. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total number of bytes a serialized Particle occupies within a buffer. Useful for advancing a buffer's read index. |
| clone() | Particle | O(N) | Creates a deep-enough copy of the Particle object. The animationFrames map is re-created. |

*Complexity O(N) refers to the size of variable-length fields (e.g., string length, map size).*

## Integration Patterns

### Standard Usage

The canonical use case is to validate and then deserialize a Particle definition from an incoming network buffer.

```java
// In a network packet handler, where 'buffer' is a received ByteBuf
// and 'readIndex' points to the start of the Particle data.

ValidationResult result = Particle.validateStructure(buffer, readIndex);

if (!result.isOk()) {
    // Disconnect client or log error; do not proceed.
    throw new ProtocolException("Invalid Particle structure: " + result.getErrorMessage());
}

Particle particleDefinition = Particle.deserialize(buffer, readIndex);

// Pass the definition to the client's particle simulation system.
particleSystem.spawnEffect(particleDefinition);

// Advance the buffer's reader index.
int bytesConsumed = Particle.computeBytesConsumed(buffer, readIndex);
buffer.readerIndex(buffer.readerIndex() + bytesConsumed);
```

### Anti-Patterns (Do NOT do this)

-   **Ignoring Validation:** Never call `deserialize` on a buffer received from an untrusted source without first calling `validateStructure`. Bypassing this check exposes the application to malformed packets that can trigger exceptions, buffer overflows, or denial-of-service vulnerabilities.
-   **Sharing Instances Across Threads:** Do not pass a reference to a Particle object to another thread. Its mutable nature will lead to race conditions. If another thread requires the data, provide it with a deep copy by calling `clone`.
-   **Long-Term Storage:** Avoid holding references to Particle objects for extended periods. They are designed as transient data carriers. If a particle definition needs to be cached, it should be stored in a more permanent, immutable format managed by a dedicated asset system.

## Data Pipeline

The Particle class is a key stage in the data flow from the network to the renderer. It transforms a raw byte stream into a structured, usable object for game systems.

> Flow:
> Raw TCP Stream -> Netty ByteBuf -> **Particle.deserialize** -> **Particle Object** -> Particle Simulation System -> Render Command Queue -> GPU

