---
description: Architectural reference for ParticleSpawner
---

# ParticleSpawner

**Package:** com.hypixel.hytale.protocol
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class ParticleSpawner {
```

## Architecture & Concepts
The ParticleSpawner class is a Data Transfer Object (DTO) that serves as the canonical, in-memory representation of a particle effect spawner's properties. It exists primarily within the network protocol layer to facilitate the serialization and deserialization of particle effect data between the server and client.

This class is not a service or a manager; it holds no logic related to the actual simulation or rendering of particles. Its sole responsibility is to act as a structured container for data that is being read from or written to a network buffer (Netty ByteBuf).

The binary layout defined by its serialization logic is highly optimized for network performance. It employs a hybrid structure consisting of:
1.  **A Nullable Bit Field:** A 2-byte header that acts as a bitmask to indicate the presence or absence of optional, nullable fields. This avoids wasting network bandwidth on empty data.
2.  **A Fixed-Size Block:** A 131-byte block containing primitives and fixed-size composite objects. This allows for extremely fast, direct-offset reads.
3.  **A Variable-Size Block:** A block containing variable-length data such as strings and arrays. The fixed-size block contains 4-byte integer offsets pointing to the start of each variable field within this block, enabling random access and preventing a full sequential scan.

This design pattern is critical for high-performance network protocols where packet structure must be predictable yet flexible.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1.  **Inbound:** The static factory method `deserialize` is invoked by a network protocol decoder when an incoming packet containing particle spawner data is being parsed.
    2.  **Outbound:** Game logic on the server instantiates a ParticleSpawner using its constructor (`new ParticleSpawner(...)`) to define a new particle effect that needs to be synchronized with clients.
- **Scope:** A ParticleSpawner instance is ephemeral and has a very short lifecycle. It typically exists only within the scope of a single network packet's processing pipeline or a single game tick's logic. It is created, its data is used by another system (like the FX renderer), and it is then immediately eligible for garbage collection.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The state of a ParticleSpawner is entirely mutable. All of its fields are public, allowing for direct, unchecked modification. This design prioritizes performance and ease of use within the controlled environment of the network and game logic threads over strict encapsulation.
- **Thread Safety:** This class is **not thread-safe**. Its public mutable fields make it inherently unsafe for concurrent access.

    **WARNING:** Modifying a ParticleSpawner instance from one thread while it is being read or serialized by another will result in undefined behavior, including data corruption, race conditions, and serialization errors. All interactions with an instance must be confined to a single thread or protected by external synchronization mechanisms.

## API Surface
The primary contract of this class is its data structure (public fields) and its serialization/deserialization interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ParticleSpawner | O(N) | Constructs a new ParticleSpawner by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. Throws ProtocolException on constraint violations. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a safe, read-only check of the binary data in a buffer to ensure it represents a valid ParticleSpawner. Does not perform a full deserialization. |
| computeSize() | int | O(N) | Calculates the total number of bytes this object will consume when serialized. Crucial for pre-allocating buffers. |
| clone() | ParticleSpawner | O(N) | Performs a deep copy of the object and its nested structures. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network layer for decoding or by game logic for encoding.

**Decoding an incoming packet:**
```java
// In a Netty ChannelInboundHandler or similar context
ByteBuf packetData = ...;
int spawnerOffset = ...; // Offset where spawner data begins

// Validate before deserializing to prevent errors from malformed packets
ValidationResult result = ParticleSpawner.validateStructure(packetData, spawnerOffset);
if (!result.isValid()) {
    throw new CorruptedPacketException(result.error());
}

ParticleSpawner spawner = ParticleSpawner.deserialize(packetData, spawnerOffset);

// Pass the DTO to the game engine's FX system
fxSystem.spawnEffectFromDefinition(spawner);
```

**Encoding an outgoing effect:**
```java
// In server-side game logic
ParticleSpawner spawner = new ParticleSpawner();
spawner.id = "magic_missile_trail";
spawner.lifeSpan = 5.0f;
// ... set other properties

// Serialize into a network buffer
ByteBuf outBuffer = PooledByteBufAllocator.DEFAULT.buffer();
spawner.serialize(outBuffer);
networkManager.sendPacket(outBuffer);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not hold references to ParticleSpawner instances for longer than necessary. They are transient data containers, not long-lived stateful objects. Caching them is unsafe due to their mutability.
- **Concurrent Modification:** Never share a ParticleSpawner instance between threads without explicit and robust locking. The class is not designed for this and will fail unpredictably.
- **Manual Serialization:** Do not attempt to read or write the binary format manually. Always use the provided `serialize` and `deserialize` methods to ensure correctness and forward compatibility. The binary layout is complex and subject to change.

## Data Pipeline
The ParticleSpawner acts as a data record that flows through the network and rendering systems.

**Inbound Flow (Client-Side):**
> Network ByteBuf -> Protocol Decoder -> **ParticleSpawner.deserialize** -> ParticleSpawner Instance -> FX System -> Particle Simulation & Rendering

**Outbound Flow (Server-Side):**
> Game Event -> Game Logic creates **ParticleSpawner Instance** -> Protocol Encoder -> **particleSpawner.serialize** -> Network ByteBuf -> Client

