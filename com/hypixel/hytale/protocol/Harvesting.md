---
description: Architectural reference for the Harvesting network data structure.
---

# Harvesting

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class Harvesting {
```

## Architecture & Concepts

The Harvesting class is a Data Transfer Object (DTO) that represents the on-the-wire binary format for data related to in-game harvesting events. It is not a service or manager, but rather a passive data structure that defines the contract between the client and server for this specific type of information.

Its primary architectural function is to provide a high-performance, low-overhead mechanism for serializing and deserializing harvesting data. To achieve this, it implements a custom binary protocol with a specific memory layout:

1.  **Nullable Bit Field:** A single leading byte acts as a bitmask. Each bit corresponds to a nullable field (itemId, dropListId), indicating whether its data is present in the payload. This avoids sending unnecessary data for optional fields.
2.  **Fixed-Size Header:** A block of a predetermined size (8 bytes) immediately follows the null bit field. This block contains integer offsets pointing to the location of variable-length data within the payload.
3.  **Variable-Size Data Block:** All variable-length data, such as strings, is appended at the end of the structure. The offsets in the fixed-size header allow for direct memory access to each field without needing to parse preceding variable data.

This layout is highly optimized for network I/O, enabling efficient validation and deserialization by allowing parsers to read the header and immediately know the full size and structure of the payload.

## Lifecycle & Ownership

As a transient DTO, Harvesting objects have an extremely short and well-defined lifecycle.

-   **Creation:**
    -   **Inbound (Receiving):** An instance is created exclusively by the static factory method `deserialize` when a network protocol decoder identifies an incoming packet of this type.
    -   **Outbound (Sending):** An instance is created directly via `new Harvesting(...)` by game logic that needs to transmit harvesting information to a remote peer.

-   **Scope:** Session-transient. An instance exists only for the immediate processing of a single network packet. It is created, its data is read or written, and it is then immediately discarded. **Warning:** Holding long-term references to Harvesting objects is a memory leak anti-pattern.

-   **Destruction:** Managed entirely by the Java Garbage Collector. Once all references are dropped after packet processing, the object becomes eligible for collection. It holds no native resources requiring explicit cleanup.

## Internal State & Concurrency

-   **State:** The internal state is fully mutable via public fields (`itemId`, `dropListId`). This design prioritizes raw performance and ease of access for serialization logic over encapsulation. The object is a simple data container.

-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. It is designed to be created, populated, and processed within the confines of a single thread, typically a Netty I/O worker or the main game thread.

    **Warning:** Sharing a Harvesting instance across multiple threads without external, user-managed synchronization will result in data corruption and non-deterministic behavior.

## API Surface

The public API is focused entirely on serialization, deserialization, and size calculation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static Harvesting | O(N) | Constructs a Harvesting object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. N is the total length of string data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. N is the total length of string data. |
| computeSize() | int | O(N) | Calculates the total number of bytes this object will consume when serialized. N is the total length of string data. |
| computeBytesConsumed(buf, offset) | static int | O(1) | Reads the header from a buffer to calculate the total size of a serialized object without performing a full deserialization. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a lightweight, non-allocating check of the binary structure in a buffer to ensure offsets and lengths are valid. |

## Integration Patterns

### Standard Usage

The class is used as part of the network protocol layer.

**Outbound (Sending Data):**
```java
// 1. Game logic creates and populates the object.
Harvesting harvestData = new Harvesting("hytale:stone", "droplist_stone_common");

// 2. The object is passed to a protocol encoder.
// (Conceptual - actual implementation is in a higher-level encoder)
ByteBuf buffer = ...;
harvestData.serialize(buffer);
networkChannel.writeAndFlush(buffer);
```

**Inbound (Receiving Data):**
```java
// 1. A protocol decoder receives a buffer for a specific packet type.
ByteBuf incomingBuffer = ...;
int offset = ...; // Start of the Harvesting data

// 2. The static deserialize method is used as a factory.
Harvesting harvestData = Harvesting.deserialize(incomingBuffer, offset);

// 3. Data is extracted and used to update game state.
String item = harvestData.itemId;
gameState.processHarvest(item);
```

### Anti-Patterns (Do NOT do this)

-   **Long-Term Storage:** Do not store Harvesting instances in game state components or caches. They are transient. Copy their data into your own domain objects for long-term use.
-   **Cross-Thread Modification:** Do not deserialize an object on a network thread and then pass the same instance to a game logic thread for modification. This is a classic race condition. Either make the object immutable after creation or copy its data.
-   **Manual Buffer Manipulation:** Do not attempt to manually read or write the binary format. The layout is complex, involving offsets and bitmasks. Always use the provided `serialize` and `deserialize` methods to guarantee protocol compliance.

## Data Pipeline

The Harvesting object is a critical link in the network data flow for harvesting events.

**Outbound Flow (e.g., Server to Client):**
> Game State Change -> **new Harvesting(...)** -> Protocol Encoder (`serialize`) -> Netty ByteBuf -> Network Socket

**Inbound Flow (e.g., Client from Server):**
> Network Socket -> Netty ByteBuf -> Protocol Decoder (`deserialize`) -> **Harvesting Instance** -> Game Event Bus -> Game State Update

