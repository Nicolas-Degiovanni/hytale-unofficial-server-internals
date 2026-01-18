---
description: Architectural reference for BlockGathering
---

# BlockGathering

**Package:** com.hypixel.hytale.protocol
**Type:** Data Structure

## Definition
```java
// Signature
public class BlockGathering {
```

## Architecture & Concepts
The BlockGathering class is a pure data structure that defines the wire format for a composite network message. It is not a service or manager; its sole purpose is to represent a schema for data related to player interactions with blocks, specifically breaking, harvesting, and interacting with soft blocks.

Architecturally, it serves as a Data Transfer Object (DTO) within the network protocol layer. It aggregates three potentially nullable sub-components: BlockBreaking, Harvesting, and SoftBlock.

The core design centers on a highly efficient, custom serialization format optimized for network performance. The binary layout consists of a fixed-size header followed by a variable-sized data block.

-   **Nullable Bit Field:** The first byte of the serialized data is a bitmask. Each bit corresponds to one of the nullable fields (breaking, harvest, soft), indicating whether its data is present in the payload. This avoids wasting space for null objects.
-   **Offset-Based Layout:** The header contains 4-byte integer offsets for each potential sub-component. These offsets are relative to the start of the variable data block. This design allows a parser to quickly seek to the data for a specific field without needing to parse preceding fields, which is critical for performance and forward compatibility.

This structure is a common pattern in high-performance networking to handle messages with optional or variable-length fields without the overhead of more descriptive formats like JSON or XML.

## Lifecycle & Ownership
-   **Creation:** BlockGathering instances are created in two primary scenarios:
    1.  **Inbound:** The static factory method `deserialize` is invoked by the network protocol dispatcher when an incoming packet of the corresponding type is received. This creates a new BlockGathering object from a raw ByteBuf.
    2.  **Outbound:** Game logic instantiates a BlockGathering object using its constructor (`new BlockGathering(...)`) to prepare a message for sending. The instance is then passed to a serializer which calls the `serialize` method.

-   **Scope:** An instance is extremely short-lived and transaction-scoped. It exists only for the duration of processing a single network packet or constructing one for transmission. It does not persist between ticks or network events.

-   **Destruction:** The object is managed by the Java Garbage Collector. Once the network packet handler or game logic that created it completes its operation, the instance becomes unreachable and is subsequently reclaimed. There are no native resources or manual cleanup procedures required.

## Internal State & Concurrency
-   **State:** The class is a mutable data container. Its public fields can be modified after instantiation. It holds no other internal state, caches, or connections to other systems.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and read within a single thread, typically a Netty I/O worker thread for deserialization or the main game thread for serialization. Concurrent modification from multiple threads will result in a corrupted state and is strictly unsupported. All synchronization must be handled externally by the calling system.

## API Surface
The public API is focused entirely on serialization, deserialization, and validation of the network format.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | BlockGathering | O(N) | **Static Factory.** Constructs a BlockGathering instance by reading from a ByteBuf at a given offset. |
| serialize(buf) | void | O(N) | Encodes the object's state into the provided ByteBuf according to the defined binary format. |
| computeSize() | int | O(N) | Calculates the total byte size required to serialize the current object state. |
| computeBytesConsumed(buf, offset) | int | O(N) | **Static.** Reads a serialized object from a buffer and returns its total size in bytes without full deserialization. |
| validateStructure(buffer, offset) | ValidationResult | O(N) | **Static.** Performs a structural integrity check on the binary data in a buffer without creating an object. |

*N = total size of the object and its sub-objects in bytes.*

## Integration Patterns

### Standard Usage
A higher-level protocol handler receives a raw network buffer and uses the static `deserialize` method to construct the object for use by game logic.

```java
// Executed within a network pipeline handler
void handlePacket(ByteBuf packetBuffer) {
    // Validate the buffer to prevent parsing invalid data
    ValidationResult result = BlockGathering.validateStructure(packetBuffer, 0);
    if (!result.isValid()) {
        // Disconnect client or log error
        throw new ProtocolException(result.error());
    }

    // Deserialize into a usable object
    BlockGathering gatheringData = BlockGathering.deserialize(packetBuffer, 0);

    // Pass the structured data to the game engine
    gameLogic.processBlockGathering(player, gatheringData);
}
```

### Anti-Patterns (Do NOT do this)
-   **Instance Reuse:** Do not reuse a BlockGathering instance across multiple network messages. The object is cheap to create and should be instantiated for each distinct operation to prevent state leakage.
-   **Multi-threaded Access:** Do not share a BlockGathering instance between threads without explicit and robust locking. The object is not designed for concurrent access.
-   **Manual Serialization:** Avoid calling `serialize` directly unless you are implementing a custom network transport. The engine's network layer is responsible for buffer management and dispatch. Incorrectly using `serialize` can lead to memory leaks or buffer corruption.

## Data Pipeline
The BlockGathering class is a critical link in the data flow between the raw network stream and the structured game logic.

> **Inbound Flow (Client to Server):**
> Raw ByteBuf -> Netty Channel Pipeline -> Protocol Dispatcher -> **BlockGathering.deserialize** -> Game Event Bus -> World Logic

> **Outbound Flow (Server to Client):**
> World Event -> Game Logic creates `new BlockGathering()` -> Protocol Dispatcher -> **BlockGathering.serialize** -> Netty Channel Pipeline -> Raw ByteBuf

