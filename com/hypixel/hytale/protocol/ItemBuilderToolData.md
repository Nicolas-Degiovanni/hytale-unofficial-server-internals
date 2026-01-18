---
description: Architectural reference for ItemBuilderToolData
---

# ItemBuilderToolData

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ItemBuilderToolData {
```

## Architecture & Concepts
ItemBuilderToolData is a data transfer object (DTO) that defines the binary wire format for builder tool configurations within the Hytale network protocol. It is not a service or a manager; its sole purpose is to act as a structured representation of data for serialization to, and deserialization from, a network byte stream.

The class encapsulates a highly optimized binary layout designed for network efficiency. The structure consists of two main parts:
1.  **Fixed-Size Header (9 bytes):** This block contains metadata about the payload.
    -   **Nullable Bit Field (1 byte):** A bitmask indicating which of the variable-sized fields are present in the payload. This avoids transmitting data for null fields.
    -   **Field Offsets (8 bytes):** Two 4-byte little-endian integers. Each integer specifies the starting offset of a variable-sized field's data, relative to the end of the fixed-size header.
2.  **Variable-Size Data Block:** This block contains the actual payload data for the non-null fields, such as the UI strings and tool states.

This architectural pattern allows for extremely fast parsing. A reader can immediately determine which fields are present and jump directly to their data using the offsets, without needing to parse the entire payload sequentially.

## Lifecycle & Ownership
-   **Creation:**
    -   **Inbound (Deserialization):** Instances are created exclusively by the static factory method `deserialize`. This is typically invoked by a higher-level network packet codec or handler within the Netty pipeline when a corresponding packet arrives.
    -   **Outbound (Serialization):** Instances are instantiated directly via `new ItemBuilderToolData()` by game logic that needs to transmit builder tool state. The public fields are then populated before the object is passed to a serializer.
-   **Scope:** The lifetime of an ItemBuilderToolData instance is exceptionally short and tied to the processing of a single network packet. It is created, its data is read or written, and it is then immediately eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or manual cleanup procedures associated with this object.

## Internal State & Concurrency
-   **State:** The object's state is fully mutable through its public fields `ui` and `tools`. This is a deliberate design choice for a DTO, simplifying the process of constructing an object before serialization. The class holds no derived or cached state; it is a direct representation of the wire format data.
-   **Thread Safety:** **This class is not thread-safe.** Instances are designed to be confined to a single thread, typically a Netty I/O worker. Sharing an instance across threads without explicit, external synchronization will result in race conditions and data corruption. The static utility methods (`deserialize`, `validateStructure`, `computeBytesConsumed`) are safe to call from any thread, provided the underlying ByteBuf is not being concurrently modified, which aligns with standard Netty practices.

## API Surface
The public API is divided between static utility methods for operating on raw buffers and instance methods for managing a specific object's state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static ItemBuilderToolData | O(N) | **Primary Factory.** Constructs an object by parsing data from a buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's current state into the provided buffer according to the defined binary format. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a safe, read-only check of the buffer to ensure the data appears valid before attempting a full deserialization. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the total size of a serialized object within a buffer without allocating a new object. Used for skipping over data. |
| computeSize() | int | O(N) | Calculates the required buffer size to serialize the current instance. |
| clone() | ItemBuilderToolData | O(N) | Creates a deep copy of the object and its contents. |

## Integration Patterns

### Standard Usage
ItemBuilderToolData is never used in isolation. It is always created by or passed to a component in the network layer.

```java
// Inbound: Deserializing from a parent packet's payload
// WARNING: The parent packet and its payload buffer are hypothetical.
ByteBuf payload = parentPacket.getPayload();

// Always validate untrusted data before parsing to prevent exceptions.
ValidationResult result = ItemBuilderToolData.validateStructure(payload, 0);
if (!result.isValid()) {
    throw new ProtocolException("Invalid ItemBuilderToolData: " + result.error());
}

ItemBuilderToolData toolData = ItemBuilderToolData.deserialize(payload, 0);
// ... process toolData in game logic
```

```java
// Outbound: Serializing data to be sent
ItemBuilderToolData toolData = new ItemBuilderToolData();
toolData.ui = new String[]{"tool_select", "color_picker"};
toolData.tools = new BuilderToolState[]{ createDefaultToolState() };

// The buffer is typically provided by a higher-level packet serializer.
ByteBuf out = ...;
toolData.serialize(out);
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not hold onto an instance of ItemBuilderToolData after it has been processed. These objects are cheap to create and should be treated as disposable. Reusing an instance from a previous packet to populate a new one can lead to subtle bugs.
-   **Cross-Thread Access:** Never pass an instance to another thread for modification. If data needs to be processed by a different thread (e.g., the main game thread), create a new, thread-safe representation of that data. Do not pass the raw protocol object.
-   **Ignoring Validation:** Calling `deserialize` on a buffer received from a remote client without first calling `validateStructure` is a security risk. A malformed buffer could cause a ProtocolException that, if unhandled, could disconnect the client or crash the worker thread.

## Data Pipeline
ItemBuilderToolData serves as the translation layer between raw bytes on the network and a structured Java object for the game logic.

> **Outbound Flow (Serialization):**
> Game Logic -> `new ItemBuilderToolData()` -> `serialize(ByteBuf)` -> Netty Protocol Encoder -> TCP Socket

> **Inbound Flow (Deserialization):**
> TCP Socket -> Netty Protocol Decoder -> `ByteBuf` -> **`ItemBuilderToolData.deserialize(ByteBuf)`** -> Game Logic Handler

