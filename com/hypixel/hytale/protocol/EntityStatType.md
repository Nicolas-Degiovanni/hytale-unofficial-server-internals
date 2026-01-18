---
description: Architectural reference for EntityStatType
---

# EntityStatType

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Model

## Definition
```java
// Signature
public class EntityStatType {
```

## Architecture & Concepts

The EntityStatType class is a data model representing a single, configurable statistic for a game entity, such as health, mana, or speed. It is a fundamental component of the Hytale network protocol layer, acting as a Data Transfer Object (DTO) for serializing and deserializing entity state between the client and server.

Its primary architectural function is to define a precise and efficient **binary wire format**. The serialization strategy is highly optimized for network performance, employing a hybrid layout of fixed-size and variable-size data blocks.

1.  **Null Bitmask:** The first byte of the serialized data is a bitmask (`nullBits`). Each bit corresponds to a nullable field (id, minValueEffects, maxValueEffects), indicating whether that field is present in the data stream. This avoids wasting bytes for null values.
2.  **Fixed-Size Block:** A 14-byte fixed-size block immediately follows the null bitmask. It contains primitive types like floats and the enum value, which have a constant, predictable size. This allows for extremely fast, direct-offset reads of core data.
3.  **Variable-Size Block:** Following the fixed block are three 4-byte integer slots that store offsets. These offsets point to the location of variable-length data (like the string *id* and nested *EntityStatEffects* objects) within a subsequent data block. This design allows the fixed-size data to be read and processed without needing to parse the entire object, which is critical for performance-sensitive network decoders.

This structure ensures that the protocol is both space-efficient and computationally inexpensive to parse, which is essential for a real-time game engine.

## Lifecycle & Ownership

-   **Creation:** Instances are created under two primary scenarios:
    1.  **Deserialization:** The static `deserialize` method instantiates the object when decoding an incoming network `ByteBuf`. This is the most common creation path on the receiving end of a network connection.
    2.  **Game Logic:** The server's game logic instantiates and populates an EntityStatType before serializing it for transmission to a client.
-   **Scope:** An EntityStatType instance is extremely short-lived. Its scope is confined to the processing of a single network packet or a discrete game state update. It is a transient container for data in motion.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the network handler or game logic method that created it completes. No long-term references are maintained by the engine.

## Internal State & Concurrency

-   **State:** The class is fully **mutable**, with public fields for direct access. This design prioritizes performance over encapsulation, a common trade-off in low-level networking and DTOs. It does not cache any data; its state is a direct representation of its fields.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty I/O thread or the main game loop.
    **WARNING:** Modifying an EntityStatType instance from multiple threads without explicit, external synchronization will result in data corruption and unpredictable behavior. Do not share instances across threads.

## API Surface

The public contract is dominated by static methods for serialization, deserialization, and validation, reinforcing its role as a protocol utility.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static EntityStatType | O(N) | Constructs an object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined wire format. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a security and integrity check on the buffer data *before* attempting deserialization. |
| computeBytesConsumed(buf, offset) | static int | O(k) | Calculates the total byte size of a serialized object within a buffer without full deserialization. k is the number of non-null variable fields. |
| computeSize() | int | O(k) | Calculates the byte size the current object will occupy when serialized. k is the number of non-null variable fields. |
| clone() | EntityStatType | O(N) | Creates a deep copy of the object and its nested EntityStatEffects. |

## Integration Patterns

### Standard Usage

The class is intended to be used by the network layer for data marshalling. Direct interaction is rare outside of packet handlers or entity management systems.

```java
// Example: In a network packet decoder
// Validate the data structure before attempting to read it.
ValidationResult result = EntityStatType.validateStructure(incomingBuffer, offset);
if (!result.isValid()) {
    throw new ProtocolException("Invalid EntityStatType: " + result.error());
}

// Deserialize the object for use in game logic.
EntityStatType stat = EntityStatType.deserialize(incomingBuffer, offset);
gameWorld.getEntity(entityId).updateStat(stat);
```

### Anti-Patterns (Do NOT do this)

-   **Ignoring Validation:** Never call `deserialize` on data from an untrusted source (i.e., any network client) without first calling `validateStructure`. Failure to do so can result in buffer over-reads, exceptions, and potential Denial of Service vulnerabilities.
-   **Concurrent Access:** Do not pass an EntityStatType instance to another thread for processing. If data must be shared, create a defensive copy using the `clone` method or extract the data into a thread-safe structure.
-   **Manual Serialization:** Do not attempt to manually read or write the fields to a buffer. The binary layout is complex and relies on the precise logic within the `serialize` and `deserialize` methods. Always use the provided API.

## Data Pipeline

The EntityStatType serves as a data payload at a specific point in the network pipeline.

> **Outbound (Server -> Client):**
> Game Logic Update -> **EntityStatType (in-memory)** -> `serialize()` -> Netty ByteBuf -> Network Encoder -> TCP Socket

> **Inbound (Client <- Server):**
> TCP Socket -> Netty ByteBuf -> Network Decoder -> `validateStructure()` -> `deserialize()` -> **EntityStatType (in-memory)** -> Game Logic Update

