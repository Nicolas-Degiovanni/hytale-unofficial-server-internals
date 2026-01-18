---
description: Architectural reference for ItemTranslationProperties
---

# ItemTranslationProperties

**Package:** com.hypixel.hytale.protocol
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class ItemTranslationProperties {
```

## Architecture & Concepts

The ItemTranslationProperties class is a specialized Data Transfer Object (DTO) designed exclusively for the network protocol layer. Its primary responsibility is to define the binary representation, or *on-the-wire format*, for an item's localizable name and description. This class is not intended for use in core game logic; it serves as a data contract for serialization and deserialization of network packets.

The binary layout is a highly optimized, non-sequential format designed for performance and flexibility with optional data. It consists of three distinct parts:

1.  **Nullable Bit Field (1 byte):** A single byte at the start of the structure. Each bit acts as a flag indicating whether a corresponding nullable field (like *name* or *description*) is present in the data stream. This allows the parser to immediately know which fields to expect.
2.  **Offset Block (8 bytes):** A fixed-size block containing 32-bit integer offsets for each potential variable-length field. These offsets are relative to the start of the *Variable Data Block*. This design allows a parser to jump directly to a specific field's data without reading all preceding fields.
3.  **Variable Data Block (N bytes):** The final section of the structure containing the actual string data, encoded as VarInt-prefixed UTF-8 byte arrays.

This structure is a common pattern in high-performance networking, as it enables extremely fast validation and partial parsing. A consumer can read the fixed-size header, check which fields exist, and jump directly to the data it needs, skipping over other fields entirely.

## Lifecycle & Ownership

-   **Creation:**
    -   **Inbound:** Instantiated by a higher-level packet decoder (e.g., a hypothetical `ItemDataPacket` decoder) when the static `deserialize` method is called on a raw Netty `ByteBuf`.
    -   **Outbound:** Instantiated directly via its constructor by game logic systems when preparing data to be sent over the network.
-   **Scope:** The object's lifetime is exceptionally short and bound to the processing of a single network packet. It is a pure **transient** object.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the data has been extracted from it (inbound) or serialized into a buffer (outbound). There are no managed resources requiring explicit cleanup.

## Internal State & Concurrency

-   **State:** The internal state is **Mutable**. The public fields `name` and `description` can be modified at any time after instantiation. The class is a simple data container and performs no caching.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared between threads without external synchronization. It is designed to be created, populated, and read within the confines of a single thread, such as a Netty I/O worker or the main game thread. The static utility methods (`deserialize`, `validateStructure`) are safe to call from any thread, provided the `ByteBuf` they operate on is accessed according to Netty's threading model.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ItemTranslationProperties | O(N) | Constructs an object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf using the defined binary format. |
| computeBytesConsumed(buf, offset) | static int | O(1) | Calculates the total size of a serialized object within a buffer without performing a full deserialization. |
| computeSize() | int | O(N) | Calculates the required buffer size to serialize the current object instance. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a structural integrity check on the binary data in a buffer. **CRITICAL:** Use this on untrusted data before deserializing. |

## Integration Patterns

### Standard Usage

This class is almost exclusively used within the `serialize` and `deserialize` methods of a parent network packet. It is an implementation detail of the protocol layer.

```java
// Example: Deserializing within a parent packet
public void deserialize(ByteBuf buf) {
    // ... read other packet fields ...
    int propertiesOffset = ...;
    this.translationProperties = ItemTranslationProperties.deserialize(buf, propertiesOffset);
    // ... now use properties ...
}

// Example: Serializing within a parent packet
public void serialize(ByteBuf buf) {
    // ... write other packet fields ...
    ItemTranslationProperties props = new ItemTranslationProperties("Sword", "A sharp blade.");
    props.serialize(buf);
}
```

### Anti-Patterns (Do NOT do this)

-   **Long-Term Storage:** Do not hold references to `ItemTranslationProperties` objects in long-lived game state components. They are transient DTOs. Extract the string data and discard the object.
-   **Ignoring Validation:** Never call `deserialize` on a buffer received from a remote client without first calling `validateStructure`. Bypassing validation can expose the server to buffer overflow vulnerabilities or cause `ProtocolException` crashes from malformed packets.
-   **Cross-Thread Modification:** Do not create an instance on one thread and pass it to another thread for serialization. The object is not thread-safe and this will lead to race conditions.

## Data Pipeline

The class acts as a translation layer between a raw byte stream and a structured, in-memory representation.

> **Inbound Flow (Deserialization):**
> Netty ByteBuf -> Packet Decoder -> **ItemTranslationProperties.deserialize()** -> ItemTranslationProperties Instance -> Game Logic (Data is extracted)

> **Outbound Flow (Serialization):**
> Game Logic -> **new ItemTranslationProperties()** -> Packet Encoder -> **item.serialize(ByteBuf)** -> Netty ByteBuf

