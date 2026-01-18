---
description: Architectural reference for RequiredBlockFaceSupport
---

# RequiredBlockFaceSupport

**Package:** com.hypixel.hytale.protocol
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class RequiredBlockFaceSupport {
```

## Architecture & Concepts
The RequiredBlockFaceSupport class is a specialized Data Transfer Object (DTO) designed for high-performance network serialization and deserialization of block connection rules. It does not contain any game logic; its sole purpose is to represent a structured set of properties that define how a block face interacts with its neighbors.

This class is a critical component of the world data protocol. Its structure is heavily optimized for network efficiency, employing a custom binary format rather than standard Java serialization. The format consists of two main parts:
1.  A **fixed-size block** of 33 bytes containing primitive types and offsets.
2.  A **variable-size block** appended after the fixed block, which stores variable-length data like strings and arrays.

The fixed block contains pointers (integer offsets) to the location of variable data within the buffer. This allows for extremely fast, non-reflective parsing directly from a Netty ByteBuf. A single leading byte, a **bitfield**, is used to encode the presence or absence of nullable fields, further minimizing payload size.

This design pattern indicates a system prioritizing low-latency, high-throughput data exchange between the client and server, which is essential for transmitting large volumes of world data.

### Lifecycle & Ownership
-   **Creation:** Instances are primarily created by the static `deserialize` method when the network layer processes an incoming data buffer. They can also be instantiated directly by game logic (e.g., a world generator or asset loader) to define new block rules before serialization.
-   **Scope:** The object's lifetime is typically very short. It exists only for the duration of a single operation, such as processing a network packet or applying a block update. It is passed by reference to the relevant game systems and is then eligible for garbage collection.
-   **Destruction:** There is no manual destruction. The Java Garbage Collector reclaims the memory once the instance is no longer referenced.

## Internal State & Concurrency
-   **State:** The class is fully **mutable**, with all fields exposed as public members. This design choice prioritizes raw performance and ease of access over encapsulation, which is common for internal, performance-critical DTOs. The state is a direct representation of the data received from the network or intended for transmission.
-   **Thread Safety:** This class is **not thread-safe**. Its mutable public fields make it inherently unsafe for concurrent modification or read/write access from multiple threads. All operations on an instance must be synchronized externally or confined to a single thread, such as a Netty event loop thread or the main game thread.

## API Surface
The public contract is dominated by static methods for serialization and validation, reinforcing its role as a protocol utility.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | RequiredBlockFaceSupport | O(N) | **Primary entry point.** Constructs an object by reading from a ByteBuf at a given offset. N is the size of the variable data. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the custom binary protocol. N is the size of the variable data. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a series of checks on a buffer to ensure it contains a valid representation of the object without performing a full deserialization. Critical for security and stability. |
| computeBytesConsumed(ByteBuf, int) | int | O(N) | Calculates the total number of bytes the object occupies within a buffer, including its variable-length fields. Used by higher-level parsers to advance buffer read positions. |
| computeSize() | int | O(N) | Calculates the number of bytes required to serialize the current object state. Useful for pre-allocating buffers. |
| clone() | RequiredBlockFaceSupport | O(N) | Performs a deep copy of the object, including its internal arrays. |

## Integration Patterns

### Standard Usage
The canonical use case is deserializing data from a network buffer provided by a higher-level packet handler. The resulting object is then passed to game systems.

```java
// In a packet handler or network processing layer
// Assume 'buffer' is a ByteBuf containing one or more objects

ValidationResult result = RequiredBlockFaceSupport.validateStructure(buffer, offset);
if (!result.isOk()) {
    throw new ProtocolException("Invalid RequiredBlockFaceSupport: " + result.getMessage());
}

RequiredBlockFaceSupport faceSupport = RequiredBlockFaceSupport.deserialize(buffer, offset);
int bytesRead = RequiredBlockFaceSupport.computeBytesConsumed(buffer, offset);
offset += bytesRead;

// Pass the deserialized object to the game engine
world.updateBlockFaceRules(position, faceSupport);
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Validation:** Never call `deserialize` without first calling `validateStructure`. Deserializing untrusted, unvalidated data can lead to buffer overflows, incorrect memory access, and server crashes.
-   **Concurrent Modification:** Do not share an instance between threads. If data must be passed across a thread boundary, create a deep copy using the `clone` method to prevent race conditions.
-   **Incorrect Offset Management:** The `offset` parameter in static methods is absolute to the buffer's beginning. Failure to correctly calculate and advance the offset when reading a stream of these objects will result in data corruption.

## Data Pipeline
This class serves as a model for data moving between its raw binary representation and its in-memory object representation.

> **Inbound Flow (Client/Server Receiving Data):**
> Raw ByteBuf from Network -> **RequiredBlockFaceSupport.validateStructure** -> **RequiredBlockFaceSupport.deserialize** -> In-Memory RequiredBlockFaceSupport Object -> Game Logic (World, Block System)

> **Outbound Flow (Client/Server Sending Data):**
> Game Logic -> new RequiredBlockFaceSupport(...) -> **instance.serialize(ByteBuf)** -> Raw ByteBuf to Network

