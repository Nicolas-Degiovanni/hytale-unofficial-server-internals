---
description: Architectural reference for ContextMenuItem
---

# ContextMenuItem

**Package:** com.hypixel.hytale.protocol.packets.worldmap
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ContextMenuItem {
```

## Architecture & Concepts
The ContextMenuItem class is a fundamental Data Transfer Object (DTO) within the Hytale network protocol. It represents a single, actionable entry within a user interface context menu, such as an item that appears when a player right-clicks on an entity or object in the world map.

This class is not a service or manager; it is a pure data container. Its primary architectural role is to define the contract and binary layout for transmitting menu item data between the server and client. The class encapsulates the serialization and deserialization logic for its own data format, a common pattern in the Hytale protocol layer to ensure that the data structure and its network representation are tightly coupled and consistent.

The binary format is highly optimized for network performance. It utilizes a fixed-size header block that contains relative offsets to a variable-data block. This allows for efficient, non-linear parsing and validation of the data stream. String fields are encoded using a variable-length integer (VarInt) prefix for their length, minimizing payload size.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary scenarios:
    1.  **Deserialization:** The static factory method `deserialize` is invoked by a higher-level packet parser (e.g., a packet representing the entire world map UI) when decoding an incoming network buffer. This is the most common creation path on the client.
    2.  **Direct Instantiation:** Game logic on the server instantiates this class via its constructor (`new ContextMenuItem(...)`) to populate a parent packet before it is sent to the client.

- **Scope:** The lifetime of a ContextMenuItem instance is extremely short and tied to a single network transaction. It is created, its data is read or written, and it is then immediately eligible for garbage collection. It is not intended to be cached or persisted.

- **Destruction:** The object is managed entirely by the Java Garbage Collector. There are no manual resource management or cleanup methods.

## Internal State & Concurrency
- **State:** The internal state consists of two mutable String fields: `name` and `command`. While technically mutable, instances should be treated as immutable after their initial creation or deserialization. Modifying the state after the fact can lead to desynchronization between the object's state and its serialized representation.

- **Thread Safety:** This class is **not thread-safe**. It is a simple data holder and contains no synchronization mechanisms. All operations on an instance must be confined to a single thread, which is typically guaranteed by the network processing pipeline (e.g., a Netty event loop thread).

## API Surface
The public API is dominated by static methods for serialization, reflecting its role as a protocol-defined structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static ContextMenuItem | O(N) | Constructs an instance by reading from a network buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the instance's state into the provided network buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a low-cost structural validation of the binary data in a buffer without full deserialization. Critical for security and stability. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will occupy when serialized. |
| computeBytesConsumed(ByteBuf, int) | static int | O(1) | Calculates the size of a serialized object directly from a buffer without creating an instance. |

*N = total length of string data*

## Integration Patterns

### Standard Usage
A ContextMenuItem is never used in isolation. It is always created or processed as part of a larger packet's serialization or deserialization routine. Developers should never call `deserialize` directly unless implementing a parent packet parser.

```java
// Hypothetical deserialization within a parent packet
// WARNING: This is a conceptual example. Do not call this directly.

// Assume 'buffer' is a ByteBuf containing the parent packet's data
// and 'offset' points to the start of a ContextMenuItem
try {
    ContextMenuItem menuItem = ContextMenuItem.deserialize(buffer, offset);
    uiManager.addContextAction(menuItem.name, menuItem.command);
} catch (ProtocolException e) {
    // Handle corrupted packet data
    connection.disconnect("Invalid context menu data");
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Serialization:** Do not modify the `name` or `command` fields after the object has been passed to a serializer. The serialized data will not reflect the change.
- **Instance Caching or Reuse:** Do not hold references to ContextMenuItem instances beyond the immediate scope of packet processing. They are transient and should be recreated as needed.
- **Cross-Thread Access:** Never access an instance from a different thread than the one that created it without explicit external synchronization. This will lead to memory visibility issues and race conditions.

## Data Pipeline
The ContextMenuItem serves as a data record that flows through the network stack. The typical flow on the receiving end (client) is as follows.

> Flow:
> Raw TCP Stream -> Netty ByteBuf -> Parent Packet Deserializer -> **ContextMenuItem.deserialize()** -> Game Logic (UI System) -> Rendered UI Element

