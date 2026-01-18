---
description: Architectural reference for EditorBlocksChange
---

# EditorBlocksChange

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class EditorBlocksChange implements Packet {
```

## Architecture & Concepts
The EditorBlocksChange packet is a specialized Data Transfer Object (DTO) designed for high-throughput world modification messages, primarily within the game's creative or editor modes. It encapsulates a batch of block and fluid changes for a specific selection area, allowing the server and client to synchronize large-scale edits efficiently.

Its primary architectural characteristic is a highly optimized, custom binary format. Unlike simpler packets, it does not serialize fields in a linear fashion. Instead, it employs a hybrid fixed-size and variable-size layout:
1.  **Nullable Field Bitmask:** The first byte of the payload is a bitmask indicating which of the nullable, variable-sized fields (like blocksChange and fluidsChange) are present in the data stream. This avoids wasting bytes for absent optional data.
2.  **Fixed-Size Header:** A block of fixed-size fields (e.g., blocksCount, advancedPreview) follows the bitmask.
3.  **Variable Data Offsets:** The header contains integer offsets that point to the location of variable-sized data blocks (the arrays) later in the payload.
4.  **Variable Data Block:** All variable-sized data is appended at the end of the packet. This layout prevents a single large array from shifting the position of all subsequent fields, simplifying parsing logic.

This structure is critical for performance, minimizing payload size for a message type that can be sent frequently and carry substantial amounts of data.

## Lifecycle & Ownership
-   **Creation:**
    -   **Sending Peer:** An instance is created via its constructor (`new EditorBlocksChange(...)`) by a high-level game system, such as the World Editor, in response to a user action. The object is populated with data and passed to the network serialization pipeline.
    -   **Receiving Peer:** The object is never created with its constructor. Instead, it is materialized by the network protocol decoder by calling the static factory method `EditorBlocksChange.deserialize`. This method reads from a raw ByteBuf and reconstructs the object.

-   **Scope:** This object is **transient and short-lived**. Its scope is confined to the network transaction. On the sending side, it exists only long enough to be serialized into a network buffer. On the receiving side, it exists only long enough for its data to be consumed by the relevant game system before being eligible for garbage collection.

-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the network layer has finished serializing it or the game logic has finished processing the received data. There is no manual destruction or resource cleanup required.

## Internal State & Concurrency
-   **State:** The object is a **mutable** data container. All of its fields are public and can be modified after instantiation. This design prioritizes performance and ease of use within a single-threaded context over enforced immutability. The existence of a `clone` method underscores the expectation that defensive copies may be necessary if the packet's state needs to be preserved.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit, external synchronization. It is designed to be created, populated, serialized, and deserialized within the confines of a single thread (typically a Netty event loop thread or the main game thread). Concurrent access will lead to race conditions and undefined behavior, especially during serialization or deserialization operations on a shared ByteBuf.

## API Surface
The public contract is dominated by serialization and validation logic, reflecting its role as a network packet.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided buffer according to the custom binary format. Throws ProtocolException on validation failures. |
| deserialize(ByteBuf, int) | static EditorBlocksChange | O(N) | Decodes a new object instance from the given buffer at the specified offset. Throws ProtocolException on malformed data. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a lightweight check on a buffer to determine if it likely contains a valid packet structure, without the overhead of full deserialization. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the total number of bytes the packet occupies in the buffer. Crucial for advancing the buffer's read index after processing. |
| clone() | EditorBlocksChange | O(N) | Creates a deep copy of the packet and its internal arrays. |

*Complexity O(N) where N is the total number of block and fluid changes.*

## Integration Patterns

### Standard Usage
The class is exclusively managed by the network protocol layer. A protocol decoder identifies incoming data by its packet ID and uses the static `deserialize` method to construct the object, which is then passed up to game logic handlers.

```java
// Executed within a Netty channel handler or similar network dispatcher
// The packet ID has already been read and identified as 222.
ByteBuf incomingBuffer = ...;
int packetOffset = ...; // Start of the packet data in the buffer

// Validate before full deserialization to fail fast
ValidationResult result = EditorBlocksChange.validateStructure(incomingBuffer, packetOffset);
if (!result.isOk()) {
    throw new ProtocolException("Invalid EditorBlocksChange packet: " + result.getErrorMessage());
}

// Deserialize and pass to the game engine
EditorBlocksChange packet = EditorBlocksChange.deserialize(incomingBuffer, packetOffset);
gameWorld.applyEditorChanges(packet);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation Post-Serialization:** Do not modify the state of an EditorBlocksChange object after it has been passed to the network layer for serialization. The serialization process may be asynchronous, leading to unpredictable data being sent.

-   **Reusing Instances:** Do not reuse packet instances for multiple messages. Always create a new instance for each distinct batch of changes to avoid state leakage and unexpected side effects.

-   **Manual Deserialization:** Do not attempt to read the fields from a ByteBuf manually. The binary layout is complex, involving bitmasks and calculated offsets. Always use the provided static `deserialize` method to ensure correctness.

## Data Pipeline
The EditorBlocksChange packet is a key component in the world editing data flow, acting as the carrier for batched modifications between the client and server.

> **Flow (Client to Server):**
> User performs a large edit in the game editor -> Editor System batches changes -> `new EditorBlocksChange(...)` is created -> Network Encoder calls `serialize` -> **EditorBlocksChange** (Binary Payload) -> TCP Stream -> Server Network Decoder calls `deserialize` -> Server World System applies changes.

