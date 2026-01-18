---
description: Architectural reference for ItemUtility
---

# ItemUtility

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ItemUtility {
```

## Architecture & Concepts

The ItemUtility class is a specialized Data Transfer Object (DTO) designed for high-performance serialization and deserialization of item-related properties within the Hytale network protocol. It is not a component of the core game logic; rather, it serves as a raw data container that represents the state of an item as it is transmitted between the client and server.

The core architectural pattern is a custom, highly optimized binary format that prioritizes compactness and read performance. This format deviates from standard serialization mechanisms to achieve minimal network overhead. The structure consists of two main parts:

1.  **Fixed-Size Header (11 bytes):** A consistent block at the beginning of the data.
    *   **Byte 0:** A bitmask (nullBits) indicating which of the nullable, variable-sized fields are present in the payload.
    *   **Bytes 1-2:** Two boolean flags, `usable` and `compatible`.
    *   **Bytes 3-6:** A 4-byte little-endian integer representing the offset to the `entityStatsToClear` data block.
    *   **Bytes 7-10:** A 4-byte little-endian integer representing the offset to the `statModifiers` data block.

2.  **Variable-Size Data Block:** Following the header, this section contains the actual data for the fields indicated by the nullBits mask. Using offsets in the header allows the deserializer to jump directly to the required data, enabling efficient parsing of complex, non-sequential data layouts.

This design allows the system to quickly determine the total size of an ItemUtility payload within a larger network buffer using `computeBytesConsumed` without needing to perform a full deserialization, which is critical for protocol parsers.

## Lifecycle & Ownership

-   **Creation:** Instances are almost exclusively created by the static `deserialize` factory method when a network I/O thread processes an incoming `ByteBuf`. For outgoing data, new instances are created via the constructor, populated with data, and passed to the `serialize` method.
-   **Scope:** Extremely short-lived. An ItemUtility object's lifetime is typically confined to the scope of a single network packet processing routine. It is created, its data is transferred to or from the game state, and it is then immediately eligible for garbage collection.
-   **Destruction:** Managed entirely by the JVM Garbage Collector. There are no native resources or manual cleanup procedures associated with this class.

## Internal State & Concurrency

-   **State:** The class is fundamentally a mutable data container. All fields are public, allowing for direct, low-overhead access. This design choice prioritizes performance over encapsulation, a common trade-off in performance-critical network code. The state represents a transient snapshot of item data.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded access, typically within a Netty event loop. Concurrent reads and writes to its public fields from multiple threads will result in race conditions, data corruption, and undefined behavior. If an instance must be passed between threads, access must be protected by external synchronization mechanisms or by creating a deep copy using the `clone` method.

## API Surface

The public API is centered around serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ItemUtility | O(N) | Constructs an ItemUtility instance by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the instance's state into the provided ByteBuf according to the defined binary format. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total size of a serialized ItemUtility in a buffer without performing a full deserialization. Essential for stream parsing. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs pre-flight checks on a buffer to verify data integrity (e.g., offsets, lengths) before attempting deserialization. |
| clone() | ItemUtility | O(N) | Creates a deep copy of the instance, including its internal collections. |

*Complexity O(N) refers to the size of the variable-length data fields.*

## Integration Patterns

### Standard Usage

The primary use case is within a network packet handler to decode item data from a raw byte buffer. The `validateStructure` method should be considered mandatory when processing data from an untrusted source.

```java
// In a network handler, reading from a ByteBuf
ByteBuf packetData = ...;
int readOffset = ...;

// It is strongly recommended to validate before deserializing
ValidationResult result = ItemUtility.validateStructure(packetData, readOffset);
if (!result.isOk()) {
    throw new ProtocolException("Invalid ItemUtility structure: " + result.getErrorMessage());
}

ItemUtility itemData = ItemUtility.deserialize(packetData, readOffset);

// Game logic can now safely use the deserialized data
processItemData(itemData);
```

### Anti-Patterns (Do NOT do this)

-   **State Reuse:** Do not reuse an ItemUtility instance across multiple serialization or deserialization operations. An instance should be treated as a single-use container for one packet's data. Modifying a deserialized object and expecting its state to be valid for other contexts is unsafe.
-   **Concurrent Modification:** Under no circumstances should an ItemUtility instance be shared and modified by multiple threads without explicit, external locking. This will lead to severe data integrity issues.
-   **Ignoring Validation:** Calling `deserialize` directly on a buffer without a prior call to `validateStructure` is dangerous. It exposes the system to potential `ProtocolException`s and buffer overflow errors if the incoming data is malformed or malicious.

## Data Pipeline

ItemUtility acts as a translation layer between the raw network byte stream and the structured in-memory representation of item data used by the game logic.

> Flow:
> Inbound Network ByteBuf -> Protocol Decoder -> **ItemUtility.deserialize** -> In-Memory **ItemUtility** Instance -> Game State Update

