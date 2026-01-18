---
description: Architectural reference for MaterialQuantity
---

# MaterialQuantity

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class MaterialQuantity {
```

## Architecture & Concepts
MaterialQuantity is a fundamental Data Transfer Object (DTO) that represents an item stack within the Hytale network protocol. It is not a service or manager, but rather a data structure that defines the precise binary wire format for item and quantity information exchanged between the client and server.

The design is heavily optimized for network efficiency, employing a hybrid fixed-size and variable-size layout. This structure minimizes bandwidth and reduces deserialization overhead.

-   **Fixed-Size Block:** A 9-byte block at the start of the structure holds predictable, non-nullable data like itemTag and quantity. This allows for extremely fast, direct memory access.
-   **Variable-Size Block:** String fields, such as itemId and resourceTypeId, which can vary in length, are stored in a separate data block. The fixed-size block contains offsets pointing to the location of this variable data.
-   **Null Field Optimization:** A single byte, `nullBits`, acts as a bitmask at the very beginning of the serialized data. Each bit corresponds to a nullable field (itemId, resourceTypeId). If a bit is not set, the corresponding field is considered null, and its data and offset are omitted entirely from the payload, saving significant space.

This architecture ensures that parsing the core, frequently accessed integer data is a constant-time operation, while still providing the flexibility to handle variable-length string identifiers.

## Lifecycle & Ownership
-   **Creation:** Instances are created on-demand by two primary actors:
    1.  The network protocol layer when a packet is being deserialized. The static `deserialize` method is the entry point.
    2.  Game logic (e.g., an inventory system) when preparing data to be sent over the network. The instance is then passed to a packet serializer.
-   **Scope:** Extremely short-lived and transient. An instance typically exists only for the duration of a single method scope—either during the serialization of an outbound packet or the deserialization and handling of an inbound one.
-   **Destruction:** The object is managed by the Java Garbage Collector and is reclaimed as soon as it is no longer referenced. There is no manual memory management or explicit destruction required.

**WARNING:** Do not retain references to MaterialQuantity instances across game ticks or network events. They are value objects and should be treated as immutable after creation or disposable after use.

## Internal State & Concurrency
-   **State:** The class is a mutable data container. All of its fields are public and can be modified after instantiation. It does not cache any data; it is a direct representation of its fields.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single thread, such as a Netty I/O worker thread or the main game logic thread. Concurrent modification from multiple threads without external synchronization will result in data corruption and unpredictable behavior.

## API Surface
The public API is focused exclusively on serialization, deserialization, and validation of the binary format.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | MaterialQuantity | O(N) | **[Factory]** Deserializes data from a ByteBuf into a new MaterialQuantity instance. N is the total length of string data. |
| serialize(buf) | void | O(N) | Serializes the object's state into the provided ByteBuf according to the defined wire format. |
| computeBytesConsumed(buf, offset) | int | O(1) | Calculates the total size of a serialized object within a buffer by reading headers and length prefixes, without a full parse. |
| computeSize() | int | O(N) | Calculates the number of bytes this object would occupy if serialized. Used for pre-allocating buffers. |
| validateStructure(buf, offset) | ValidationResult | O(1) | Performs a structural integrity check on the data in a buffer, verifying offsets and lengths without full deserialization. |

## Integration Patterns

### Standard Usage
MaterialQuantity is almost always used within the context of a packet serializer or deserializer. The static factory method `deserialize` is the canonical way to create an instance from network data.

```java
// Example: Inside a packet handler or deserializer
public void processInventoryUpdate(ByteBuf packetData) {
    int currentOffset = 4; // Assuming some packet header
    MaterialQuantity receivedItem = MaterialQuantity.deserialize(packetData, currentOffset);

    // Now use the fully hydrated object in game logic
    player.getInventory().addItem(receivedItem.itemId, receivedItem.quantity);
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Serialization:** Never attempt to read or write the binary format manually. The layout involving the null-bitmask and variable offsets is complex and error-prone. Always use the `serialize` and `deserialize` methods.
-   **Instance Re-use:** Do not modify a deserialized instance and attempt to re-serialize it. While technically possible, it breaks the "value object" pattern and can lead to bugs. Treat them as immutable reads or create fresh instances for writing.
-   **Cross-Thread Sharing:** Do not pass a MaterialQuantity instance from a network thread to a game logic thread without creating a defensive copy or ensuring a proper memory synchronization barrier.

## Data Pipeline
MaterialQuantity acts as the bridge between in-memory game state and the on-the-wire binary representation for item stacks.

> **Outbound Flow (Serialization):**
> Game Logic (Inventory) -> Creates **MaterialQuantity** instance -> Packet Serializer calls `serialize()` -> Netty ByteBuf -> Network Socket

> **Inbound Flow (Deserialization):**
> Network Socket -> Netty ByteBuf -> Packet Deserializer calls `deserialize()` -> Creates **MaterialQuantity** instance -> Game Logic (Packet Handler)

