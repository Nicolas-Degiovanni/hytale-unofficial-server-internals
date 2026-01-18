---
description: Architectural reference for ItemResourceType
---

# ItemResourceType

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class ItemResourceType {
```

## Architecture & Concepts
The ItemResourceType class is a fundamental data structure within the Hytale network protocol layer. It is not a service or a long-lived component; rather, it serves as a direct, in-memory representation of an item stack as defined by the binary protocol. Its primary responsibility is to facilitate the transfer of item data—specifically an item identifier and its quantity—between the client and server.

This class embodies the **Data Contract** for item information. The serialization and deserialization logic is tightly coupled to a highly optimized binary format designed to minimize network payload size. Key characteristics of this format include:

*   **Fixed-Size Block:** A 5-byte header containing a nullability bitmask and the integer quantity.
*   **Nullable Fields:** A bitmask (`nullBits`) is used to indicate the presence of optional fields, avoiding the need to transmit empty data.
*   **Variable-Length Encoding:** String lengths are encoded using VarInts, ensuring that small strings consume minimal space on the wire.

Any modification to this class, particularly its serialization methods or field layout, constitutes a breaking change to the network protocol and requires a version bump.

## Lifecycle & Ownership
-   **Creation:** Instances are ephemeral and created under two main circumstances:
    1.  **Inbound:** The network layer instantiates objects by calling the static `deserialize` factory method when decoding an incoming packet from a Netty ByteBuf.
    2.  **Outbound:** Game logic instantiates objects using a public constructor (e.g., `new ItemResourceType("hytale:stone", 64)`) when preparing data to be sent in an outgoing packet.

-   **Scope:** The lifetime of an ItemResourceType instance is extremely short. It typically exists only for the duration of a single packet processing cycle or a discrete game logic operation. It is not intended to be cached or held in long-term storage.

-   **Destruction:** Instances are managed entirely by the Java Garbage Collector. As they are simple data holders with no native resources, they are reclaimed as soon as they fall out of scope. No manual destruction or cleanup is necessary.

## Internal State & Concurrency
-   **State:** The class is **mutable**. Its public fields, `id` and `quantity`, can be directly modified after instantiation. This design prioritizes performance and ease of use by avoiding object allocation overhead for simple state changes, such as decrementing an item stack.

-   **Thread Safety:** ItemResourceType is **not thread-safe**. It contains no internal locking or synchronization primitives. It is designed for use within a single-threaded context, such as a Netty I/O thread or the main game update loop.

    **WARNING:** Sharing and concurrently modifying an ItemResourceType instance across multiple threads without explicit external synchronization will result in data corruption and undefined behavior.

## API Surface
The public API is focused on serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ItemResourceType | O(N) | Constructs an instance by reading from a ByteBuf. Throws ProtocolException on malformed data. N is the length of the string `id`. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the protocol specification. |
| computeSize() | int | O(N) | Calculates the number of bytes this object would occupy when serialized. Useful for pre-allocating buffers. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check on a buffer to ensure it contains a structurally valid object without full deserialization. Critical for security and stability. |
| clone() | ItemResourceType | O(1) | Creates a shallow copy of the object. |

## Integration Patterns

### Standard Usage
The canonical use case involves decoding an object from a network buffer, performing some game logic, and potentially encoding a new or modified object for a response.

```java
// Executed within a Netty channel handler or similar context
ByteBuf incomingPacket = ...;

// Validate before processing to prevent errors from malformed packets
ValidationResult result = ItemResourceType.validateStructure(incomingPacket, offset);
if (!result.isOk()) {
    // Disconnect client or log error
    throw new ProtocolException("Invalid ItemResourceType structure: " + result.getReason());
}

// Deserialize into a usable object
ItemResourceType receivedItem = ItemResourceType.deserialize(incomingPacket, offset);

// Game logic can now safely operate on the object
player.getInventory().addItem(receivedItem);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Reuse:** Do not hold onto a deserialized instance and modify it for later use if that instance is also referenced by other systems. This can cause state desynchronization. It is safer to `clone()` the object or create a new one.
-   **Ignoring Validation:** Bypassing `validateStructure` on data from an untrusted source (i.e., any client) is a significant security risk. A malicious client could send a malformed length field, leading to excessive memory allocation and a Denial-of-Service vulnerability.
-   **Cross-Thread Access:** Never pass an ItemResourceType instance from the network thread to a game logic thread (or vice-versa) without proper synchronization or by creating a defensive copy.

## Data Pipeline
ItemResourceType is a critical link in the data flow between the raw network stream and the server's game state.

> Flow:
> Inbound Netty ByteBuf -> **ItemResourceType.deserialize** -> Game Logic (e.g., InventoryManager) -> State Update
>
> Game Event -> Game Logic creates `new ItemResourceType()` -> **ItemResourceType.serialize** -> Outbound Netty ByteBuf

