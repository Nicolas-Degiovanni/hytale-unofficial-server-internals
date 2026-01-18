---
description: Architectural reference for ParticleSpawnerGroup
---

# ParticleSpawnerGroup

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class ParticleSpawnerGroup {
```

## Architecture & Concepts
The ParticleSpawnerGroup class is a Data Transfer Object (DTO) that defines the network contract for particle effect configurations. It is not an active component of the game engine; rather, it serves as a structured, serializable representation of all properties required to define a group of particle emitters.

Its primary role is to facilitate the transfer of complex particle data between the server and client. The class structure is a direct one-to-one mapping of a highly optimized binary layout, designed for minimal network overhead and high-speed serialization and deserialization.

The binary layout is partitioned into two distinct sections:
1.  **Fixed-Size Block:** A 121-byte block containing primitive types, fixed-size objects (like Vector3f), and offsets to the variable data section. This ensures predictable and fast reads for the most common data.
2.  **Variable-Size Block:** A subsequent block containing data of indeterminate length, such as strings and arrays. The fixed block contains pointers to the start of each variable field within this section.

A key feature is the use of a **null-bit field** at the start of the structure. This 2-byte bitmask efficiently encodes the presence or absence of nullable fields, eliminating the need to transmit empty data for optional properties.

## Lifecycle & Ownership
-   **Creation:** An instance of ParticleSpawnerGroup is created under two primary circumstances:
    1.  **Deserialization:** The static method `deserialize` is invoked by a network packet handler (typically on a Netty I/O thread) when a corresponding packet is received. This constructs the object from a raw ByteBuf.
    2.  **Manual Instantiation:** The server's game logic instantiates and populates the object when it needs to define and transmit a new particle effect to a client.

-   **Scope:** The object's lifetime is intentionally brief. It exists only for the duration of a single network transaction or as a temporary data container. It is not designed to be cached or persisted in a long-term state.

-   **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as it is no longer referenced, which is typically after the network handler has processed it or after it has been serialized into an outgoing buffer. There are no manual cleanup requirements.

## Internal State & Concurrency
-   **State:** The internal state is fully **mutable**. All fields are public, providing direct, unrestricted access. This design choice prioritizes raw performance over encapsulation, which is common in low-level networking DTOs. The object acts as a simple data container.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms.

    **WARNING:** Concurrent modification of a ParticleSpawnerGroup instance will lead to race conditions and data corruption. It must only be accessed and modified from a single thread, such as the main game thread or a specific Netty event loop thread. Do not share instances between threads without explicit external synchronization.

## API Surface
The public contract is dominated by serialization and validation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | ParticleSpawnerGroup | O(N) | **[Critical]** Static factory method. Constructs an object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Encodes the object's state into the provided ByteBuf according to the defined binary protocol. Throws ProtocolException if constraints are violated. |
| validateStructure(buf, offset) | ValidationResult | O(N) | **[Security]** Performs pre-deserialization checks on a buffer to ensure offsets and lengths are valid. Crucial for preventing crashes from malformed packets. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total number of bytes the object occupies in the buffer, accounting for all variable-length fields. |
| computeSize() | int | O(N) | Calculates the required size in bytes to serialize the current object state. Useful for buffer pre-allocation. |
| clone() | ParticleSpawnerGroup | O(N) | Performs a deep copy of the object and its contents, including nested objects and arrays. |

## Integration Patterns

### Standard Usage
The class is typically used within a network protocol handler to decode incoming data. The validation step is a mandatory precursor to deserialization to ensure network stability.

```java
// Executed within a Netty channel handler
void handlePacket(ByteBuf packetData) {
    int offset = 0; // Start of the data in the buffer

    // 1. Validate the structure before attempting to read
    ValidationResult result = ParticleSpawnerGroup.validateStructure(packetData, offset);
    if (!result.isOk()) {
        // Disconnect client or log error; do not proceed
        throw new ProtocolException("Invalid ParticleSpawnerGroup: " + result.getErrorMessage());
    }

    // 2. Deserialize into a usable object
    ParticleSpawnerGroup spawner = ParticleSpawnerGroup.deserialize(packetData, offset);

    // 3. Pass the DTO to the game engine's particle system
    game.getParticleSystem().spawnEffect(spawner);
}
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Validation:** Never call `deserialize` without first calling `validateStructure`. A malicious or malformed packet could contain invalid offsets or lengths, leading to buffer overflows or server crashes.
-   **State Reuse:** Do not modify a ParticleSpawnerGroup instance after it has been passed to a serialization routine. The serialization methods read the state directly, and any concurrent modification can result in a corrupted network packet.
-   **Long-Term Storage:** Do not hold references to these objects in long-lived caches or game state components. They are designed as transient data carriers. Convert them into an internal engine representation for storage.

## Data Pipeline
The ParticleSpawnerGroup serves as the deserialized representation of particle data within the network layer. It is the bridge between raw bytes on the wire and a structured object that the game engine can interpret.

> Flow:
> Netty ByteBuf -> **ParticleSpawnerGroup.validateStructure** -> **ParticleSpawnerGroup.deserialize** -> ParticleSpawnerGroup Instance -> Game Particle System -> Render Engine

