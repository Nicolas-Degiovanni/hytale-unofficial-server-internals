---
description: Architectural reference for BiomeData
---

# BiomeData

**Package:** com.hypixel.hytale.protocol.packets.worldmap
**Type:** Transient

## Definition
```java
// Signature
public class BiomeData {
```

## Architecture & Concepts
The BiomeData class is a high-performance Data Transfer Object (DTO) designed for network serialization. It represents the essential properties of a biome or world zone, such as its identifier, name, and color. This class is not a general-purpose data model; its design is strictly optimized for the Hytale binary network protocol.

The core architectural concept is a custom serialization format that avoids the overhead of reflection-based libraries. The binary layout is split into two main sections to handle both fixed-size and variable-size data efficiently:

1.  **Fixed-Size Block (17 bytes):** This initial block contains fields that are always present and have a predictable size. It includes a null-tracking bitfield, integer identifiers, and, critically, offsets that point to the location of variable-sized data.
2.  **Variable-Size Block:** This block is appended after the fixed-size block and contains the actual data for strings like zoneName and biomeName.

This offset-based approach allows for extremely fast, non-allocating reads during validation and deserialization, as the system can jump directly to the required data without parsing intermediate fields.

**WARNING:** The methods in this class operate directly on Netty ByteBuf instances. Misuse, such as providing incorrect offsets or modifying the buffer during a read operation, will lead to protocol corruption, client-server desynchronization, or application crashes.

## Lifecycle & Ownership
-   **Creation:** BiomeData instances are created under two primary circumstances:
    1.  **Inbound:** The static deserialize method is called by a network packet decoder when a corresponding packet is received from the network stream.
    2.  **Outbound:** Game logic on the sending side (typically a server) instantiates a new BiomeData object using its constructor, populates its fields, and passes it to a packet encoder.
-   **Scope:** The object's lifetime is transient and typically confined to the scope of a single network packet's processing cycle. It is created, processed by game logic, and then becomes eligible for garbage collection.
-   **Destruction:** The object is managed by the Java garbage collector. There are no native resources or manual cleanup procedures required.

## Internal State & Concurrency
-   **State:** The BiomeData class is a mutable container for biome properties. Its fields are public and can be modified after instantiation.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created and used within a single thread, such as a Netty event loop thread or the main game logic thread. Concurrent reads and writes from multiple threads will result in race conditions and undefined behavior. Any multi-threaded access must be managed with external synchronization.

## API Surface
The public contract is focused on serialization, deserialization, and validation against a raw byte buffer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | BiomeData | O(N) | **[Factory]** Constructs a BiomeData instance by reading from the buffer at a given offset. N is the total length of the string data. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided buffer using the custom binary protocol. N is the total length of the string data. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a read-only check of the buffer to ensure the data represents a valid BiomeData structure without full deserialization. Critical for security and stability. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will consume when serialized. Useful for pre-allocating buffers. |
| computeBytesConsumed(ByteBuf, int) | int | O(N) | Reads the structure from a buffer to determine its total size in bytes. |

## Integration Patterns

### Standard Usage
The primary use case involves either serializing an instance to send over the network or deserializing data received from the network.

```java
// Example: Server preparing to send biome information
ByteBuf buffer = Unpooled.buffer();
BiomeData biome = new BiomeData(10, "Kweebec Forest", "Forest", 0x228B22);

// The packet encoder would call this method
biome.serialize(buffer);

// The buffer is now ready to be written to the network channel.
```

### Anti-Patterns (Do NOT do this)
-   **Shared Mutable State:** Do not pass a BiomeData instance between threads. If data needs to be shared, create a deep copy using the copy constructor or an immutable representation.
-   **Ignoring Validation:** Never call deserialize on un-trusted data without first calling validateStructure. A malformed packet could provide invalid offsets or lengths, leading to buffer over-reads and server instability.
-   **Reusing Instances:** Do not reuse a BiomeData instance for a different logical biome without re-populating all its fields. Its mutable nature makes it unsafe for caching unless it is treated as immutable after creation.

## Data Pipeline
The BiomeData class is a critical link in the network data flow for world information.

> **Inbound Flow (Receiving Data):**
> Netty Channel -> ByteBuf -> Packet Decoder -> **BiomeData.deserialize()** -> BiomeData Instance -> World Map System

> **Outbound Flow (Sending Data):**
> World Generation Logic -> BiomeData Instance -> Packet Encoder -> **biomeData.serialize(ByteBuf)** -> ByteBuf -> Netty Channel

