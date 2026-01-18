---
description: Architectural reference for ResourceType
---

# ResourceType

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class ResourceType {
```

## Architecture & Concepts
The ResourceType class is a fundamental Data Transfer Object (DTO) within the Hytale network protocol layer. It is not a service or a manager; its sole purpose is to represent a simple, serializable data structure that defines a game resource by its unique identifier and an optional visual icon.

Its primary architectural significance lies in its custom, high-performance serialization format. This format is heavily optimized for network transmission, prioritizing compact size and efficient decoding. The core mechanism involves a fixed-size header containing a **null bitmask** and an **offset table**, followed by a variable-length data block.

-   **Null Bitmask:** A single byte at the start of the serialized data where each bit corresponds to a nullable field (id, icon). This allows the deserializer to immediately know which fields are present without reading further, and it avoids wasting network bandwidth on null values.
-   **Offset Table:** A series of fixed-size integers that store the starting position of each variable-length field within the subsequent data block. This enables direct, non-sequential access to any field, which is critical for forward-compatibility and efficient partial-decoding.

This design pattern is common in high-performance networking to decouple the fixed-size metadata of a structure from its variable-size payload, ensuring predictable and fast parsing of incoming data streams.

## Lifecycle & Ownership
-   **Creation:** ResourceType instances are created transiently. They are instantiated either by the network protocol engine via the static `deserialize` method when decoding an incoming packet, or directly by game logic (e.g., `new ResourceType(...)`) when preparing data for an outgoing packet.
-   **Scope:** The lifetime of a ResourceType object is extremely short. It is typically scoped to the processing of a single network packet or a single game-logic operation. These objects are not intended for long-term storage within the game state.
-   **Destruction:** Instances are managed by the Java Garbage Collector. They become eligible for collection as soon as they are no longer referenced, which is typically immediately after the parent network packet has been processed or sent. No manual resource management is required.

## Internal State & Concurrency
-   **State:** The class is a **Mutable** DTO. Its state is composed of its public `id` and `icon` String fields. There is no internal caching or complex state management; it is a pure data container. The fields can be modified directly at any time after instantiation.
-   **Thread Safety:** This class is **not thread-safe**. The public, mutable fields make it susceptible to race conditions if an instance is shared and modified concurrently across multiple threads.

    **WARNING:** ResourceType instances must only be accessed and mutated from a single thread, such as a Netty I/O worker thread or the main game update thread. Sharing instances across threads without external synchronization will lead to data corruption and unpredictable behavior.

## API Surface
The public API is focused entirely on serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ResourceType | O(N) | Constructs a ResourceType by decoding data from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Encodes the object's state into the provided ByteBuf according to the defined binary format. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total byte size of a serialized object within a buffer without full deserialization. |
| computeSize() | int | O(N) | Calculates the required byte size to serialize the current object state. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a structural integrity check on serialized data in a buffer. Does not create an object. |

*Complexity O(N) refers to the total length of the string data.*

## Integration Patterns

### Standard Usage
ResourceType is used as a data payload within larger packet structures. It is never used as a standalone service.

**Deserialization (Receiving Data):**
The network layer decodes a packet and invokes the static deserialize method to populate the object from the raw byte stream.
```java
// Executed within a network packet's decode method
// buf is the incoming ByteBuf, and offset is the start of the ResourceType data
ResourceType receivedResource = ResourceType.deserialize(buf, offset);
String resourceId = receivedResource.id;
// ... process the received resource data
```

**Serialization (Sending Data):**
Game logic creates and populates a ResourceType instance, which is then passed to a parent packet object for serialization.
```java
// Executed by game logic preparing to send a packet
ResourceType resourceToSend = new ResourceType("hytale:stone", "gui/items/stone.png");

// The parent packet's serialize method will then call:
resourceToSend.serialize(outputByteBuf);
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term State:** Do not retain ResourceType instances as part of the persistent game state. They are transport objects. Convert their data into a stable, internal domain model for long-term use.
-   **Cross-Thread Sharing:** Never pass a ResourceType instance from a network thread to a game logic thread (or vice-versa) if either thread might modify it. This is a critical concurrency violation.
-   **Manual Serialization:** Do not attempt to manually read or write the binary format. The layout, including the null bitmask and offset calculations, is complex. Always use the provided `serialize` and `deserialize` methods to ensure correctness.

## Data Pipeline
The ResourceType class is a critical link in the data serialization and deserialization pipeline. Its on-the-wire format is explicitly defined to be machine-readable and efficient.

> **Serialization Flow:**
> Game Logic creates `ResourceType` -> Parent Packet calls `resource.serialize(buf)` -> ByteBuf written to Network Socket

> **Deserialization Flow:**
> Network Socket reads to `ByteBuf` -> Parent Packet calls `ResourceType.deserialize(buf)` -> Game Logic consumes `ResourceType`

The binary layout within the ByteBuf follows a strict structure:

> **On-the-Wire Format:**
> 1.  **Null Bitmask (1 byte):** A bit field indicating which nullable fields are present. (Bit 0 for `id`, Bit 1 for `icon`).
> 2.  **Offset Table (8 bytes):** A fixed block containing two 4-byte little-endian integer offsets, one for `id` and one for `icon`. The offset is relative to the start of the variable data block. A value of -1 indicates the field is null.
> 3.  **Variable Data Block (N bytes):** The tightly packed, variable-length data for all non-null fields. Each string is prefixed with its length encoded as a VarInt.

