---
description: Architectural reference for ProjectileConfig
---

# ProjectileConfig

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class ProjectileConfig {
```

## Architecture & Concepts

The **ProjectileConfig** class is a Data Transfer Object (DTO) that defines the complete set of properties for a projectile entity within the Hytale engine. It serves as a critical component of the network protocol layer, providing a structured, in-memory representation of data that is serialized to, and deserialized from, a raw byte stream.

Architecturally, this class acts as a serialization boundary between high-level game logic and the low-level network transport. It encapsulates the complex and highly-optimized binary layout of projectile data, abstracting away manual byte manipulation from the rest of the engine.

The binary format is a high-performance hybrid model:
1.  **Nullable Bit Field:** A single byte at the start of the structure acts as a bitmask, indicating which of the nullable, complex fields are present in the data stream. This avoids wasting space for optional data.
2.  **Fixed-Size Block:** A contiguous block of 163 bytes containing primitive types like doubles and integers, which have a constant and predictable size.
3.  **Variable-Size Block:** For fields with dynamic sizes (e.g., **Model**, **Map** of interactions), the fixed block contains 4-byte integer offsets pointing to the start of their data within a variable-data region that follows the fixed block. This allows for efficient random access to fixed fields while accommodating variable-length data.

This design prioritizes minimal packet size and fast deserialization, which are critical for real-time game networking.

## Lifecycle & Ownership

-   **Creation:** An instance of **ProjectileConfig** is created in one of two primary scenarios:
    1.  **Inbound:** The static factory method **deserialize** is called by a network packet handler (e.g., a Netty channel handler) when a corresponding packet is received from the network. This is the most common creation path.
    2.  **Outbound:** Game logic instantiates a **ProjectileConfig** directly using its constructor to define a new projectile (e.g., when a player fires a bow). This instance is then passed to the network layer for serialization.
-   **Scope:** The object is transient and has a very short lifetime. It typically exists only for the duration of a single frame or network event processing cycle. It is passed by reference between systems but is not intended to be stored long-term.
-   **Destruction:** As a standard Java object with no native resources, it is managed entirely by the Java Garbage Collector. It becomes eligible for collection as soon as it is no longer referenced by the network handler or game systems that were processing it.

## Internal State & Concurrency

-   **State:** The internal state is fully **mutable**. All fields are public and can be modified directly after instantiation. The class is a simple data container and performs no internal caching or calculations beyond what is required for serialization.

-   **Thread Safety:** **WARNING:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. All fields are publicly accessible, making it susceptible to race conditions if accessed concurrently. It is designed to be created, populated, and read within a single, well-defined thread, such as a Netty event loop thread or the main game logic thread. Any cross-thread usage **must** be managed with external synchronization.

## API Surface

The public contract is focused exclusively on serialization, deserialization, and validation of the network data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ProjectileConfig | O(N) | Constructs a **ProjectileConfig** by reading from a **ByteBuf**. Throws **ProtocolException** on malformed data. N is the number of interactions. |
| serialize(buf) | void | O(N) | Writes the object's state into the given **ByteBuf** according to the defined binary protocol. Throws **ProtocolException** if constraints are violated. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check of a buffer to ensure it contains a structurally valid **ProjectileConfig**. Does not perform a full deserialization. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will consume when serialized. Useful for pre-allocating buffers. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the size of an already-serialized object directly from the buffer. |
| clone() | ProjectileConfig | O(N) | Creates a deep copy of the configuration object. |

## Integration Patterns

### Standard Usage

The primary use case involves the network layer decoding a buffer into a **ProjectileConfig** object, which is then passed to a game system for processing.

```java
// Executed within a network thread or game loop
public void handleProjectilePacket(ByteBuf packet) {
    // Validate before deserializing to prevent errors
    ValidationResult result = ProjectileConfig.validateStructure(packet, packet.readerIndex());
    if (!result.isValid()) {
        // Disconnect client or log error
        throw new ProtocolException("Invalid projectile packet: " + result.error());
    }

    ProjectileConfig config = ProjectileConfig.deserialize(packet, packet.readerIndex());

    // Pass the fully-formed DTO to the world or entity system
    world.getProjectileSystem().spawnProjectile(config);
}
```

### Anti-Patterns (Do NOT do this)

-   **Concurrent Modification:** Do not read a **ProjectileConfig** on one thread while it is being written to on another. This will lead to data corruption and unpredictable behavior.
-   **Stateful Reuse:** Do not hold onto a **ProjectileConfig** instance and modify it for reuse. These objects are cheap to create and should be treated as immutable after being passed to another system. If a template is needed, use the **clone** method to create a new, distinct copy.
-   **Ignoring Validation:** Do not call **deserialize** on untrusted data without first calling **validateStructure**. A malicious or malformed packet could throw exceptions that crash the network thread if not handled properly.

## Data Pipeline

The class is a key translation point in the data flow between the network and the game simulation.

> **Inbound Flow (Client/Server Receiving Data):**
> Raw ByteBuf from Netty -> **ProjectileConfig.validateStructure** -> **ProjectileConfig.deserialize** -> **ProjectileConfig Instance** -> Game Systems (e.g., World, EntitySpawner)

> **Outbound Flow (Client/Server Sending Data):**
> Game Event (e.g., Player Shoots Bow) -> Game Logic Creates **ProjectileConfig Instance** -> **instance.serialize(ByteBuf)** -> Raw ByteBuf sent to Netty Pipeline

