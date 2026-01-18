---
description: Architectural reference for FormattedMessage
---

# FormattedMessage

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class FormattedMessage {
```

## Architecture & Concepts
The FormattedMessage class is a core Data Transfer Object (DTO) within the Hytale network protocol. Its primary function is to represent rich, hierarchical text structures that can be efficiently serialized for network transport and deserialized for rendering in the game client. It is the backbone for features like styled chat, item descriptions, and dynamic UI text.

The design is optimized for performance and low memory overhead during network I/O. It achieves this through a custom binary format composed of two main parts:
1.  A **Fixed-Size Block** (34 bytes) containing primitive values and offsets.
2.  A **Variable-Size Block** containing all dynamic data like strings, arrays, and maps.

Offsets within the fixed block act as pointers to the start of corresponding data in the variable block. This layout allows for rapid, predictable parsing of the message structure without needing to scan the entire payload.

A key optimization is the `nullBits` byte at the start of the serialized data. This bitfield efficiently encodes the presence or absence of nullable fields, ensuring that optional data consumes zero space on the wire when not used.

The structure is inherently recursive; a FormattedMessage can contain an array of child FormattedMessage objects. This enables complex, nested text formatting, similar to an HTML DOM tree.

## Lifecycle & Ownership
-   **Creation:** Instances are primarily created by the network layer when a packet is decoded, using the static `deserialize` factory method. They can also be constructed programmatically by game logic (e.g., a chat system preparing a message for broadcast) before being passed to the serialization pipeline.
-   **Scope:** The lifetime of an instance is typically short and confined to a single logical operation, such as the handling of one network packet or the rendering of a single UI frame. It is a value object, not a managed service.
-   **Destruction:** As a plain Java object with no external resources, a FormattedMessage is eligible for garbage collection as soon as it is no longer referenced. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** The class is a mutable data container. All of its fields are public, allowing for direct, unchecked modification. This design prioritizes performance and ease of use within the single-threaded context of a network or game loop, but it carries significant risks if used improperly.
-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization.

    **WARNING:** Sharing a FormattedMessage instance across multiple threads without robust external synchronization will result in unpredictable behavior, data corruption, and race conditions. It is designed to be created, processed, and discarded within a single thread of execution.

## API Surface
The public contract is dominated by static methods for serialization and validation, reflecting its role as a protocol-level data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | FormattedMessage | O(N) | **[Primary Constructor]** Deserializes a message from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Serializes the object's state into the provided ByteBuf according to the defined binary format. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size of a serialized message within a buffer by traversing its structure. Does not allocate a FormattedMessage object. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a security and integrity check on the binary data before attempting full deserialization. Critical for preventing malformed packet exploits. |
| clone() | FormattedMessage | O(N) | Creates a deep copy of the message and its entire child hierarchy. |
| computeSize() | int | O(N) | Calculates the number of bytes this object would occupy if serialized. |

*N represents the total number of nodes and the size of data within the message's hierarchy.*

## Integration Patterns

### Standard Usage
The class is intended to be used by network packet handlers to decode incoming data or encode outgoing messages.

```java
// Example: Deserializing a message within a packet handler
public void handlePacket(ByteBuf packetData) {
    // Validate before deserializing to prevent errors with malicious data
    ValidationResult result = FormattedMessage.validateStructure(packetData, packetData.readerIndex());
    if (!result.isValid()) {
        throw new ProtocolException("Invalid FormattedMessage structure: " + result.error());
    }

    FormattedMessage message = FormattedMessage.deserialize(packetData, packetData.readerIndex());

    // Pass the immutable DTO to other systems for processing
    GameUI.renderChatMessage(message);
}
```

### Anti-Patterns (Do NOT do this)
-   **Modification of Shared Instances:** Modifying a FormattedMessage after it has been passed to another system is dangerous. If changes are needed, use the `clone` method to create a distinct copy first.
-   **Ignoring Pre-validation:** Calling `deserialize` on a buffer received from an untrusted source without first calling `validateStructure` is a security risk. A malformed packet could trigger uncaught exceptions and crash the server or client.
-   **Long-Term Storage:** This object is optimized for transit, not for long-term storage. Its mutable nature and lack of encapsulation make it a poor choice for representing persistent state. Convert it to an immutable, domain-specific model if you need to retain the data.

## Data Pipeline
FormattedMessage acts as a bridge, converting raw binary data from the network into a structured object that application-level systems like the UI can understand and process.

> Flow:
> Raw Network ByteBuf -> **FormattedMessage.validateStructure** -> **FormattedMessage.deserialize** -> In-Memory FormattedMessage Object -> UI Rendering System -> Displayed Text

