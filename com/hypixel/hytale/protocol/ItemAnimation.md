---
description: Architectural reference for ItemAnimation
---

# ItemAnimation

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class ItemAnimation {
```

## Architecture & Concepts
The ItemAnimation class is a specialized Data Transfer Object (DTO) designed for high-performance network serialization and deserialization of item animation properties. It is not a general-purpose animation controller but rather a raw data container that directly maps to a specific binary layout on the wire.

Its core architectural purpose is to act as the in-memory representation of a complex, variable-length data structure received from or sent to the network. The design prioritizes raw performance and memory efficiency over encapsulation, which is evident from its public fields and static serialization methods.

The binary protocol it represents is non-trivial, employing a hybrid layout:
1.  **Fixed-Size Block:** A 12-byte block at the beginning of the structure holds primitive types like booleans, floats, and a bitmask.
2.  **Offset Table:** A 20-byte block contains integer offsets pointing to the location of variable-length string data.
3.  **Variable-Size Block:** A subsequent data region where the actual UTF-8 encoded strings are stored.

This structure allows for efficient partial reads and validation, as the fixed-size header can be processed without reading the entire data payload. A key optimization is the `nullBits` byte, which acts as a bitmask to indicate which of the five nullable string fields are present in the payload, avoiding the need for null terminators or wasted space for empty fields.

## Lifecycle & Ownership
-   **Creation:** Instances are created under two primary scenarios:
    1.  **Inbound:** The static `deserialize` method constructs a new ItemAnimation object by reading from a Netty ByteBuf. This is the most common creation path, typically invoked by a network packet decoder.
    2.  **Outbound:** Game logic instantiates an ItemAnimation via its constructor to define an animation set that needs to be sent over the network.
-   **Scope:** An ItemAnimation object is ephemeral and has a very short lifecycle. It is intended to exist only for the duration of processing a single network packet or preparing one for transmission. Some instances may be held longer as part of a static item definition, but those created during network I/O are immediately eligible for garbage collection after the relevant logic completes.
-   **Destruction:** The object is managed by the Java Garbage Collector. There are no native resources or explicit cleanup methods required.

## Internal State & Concurrency
-   **State:** The state is fully mutable. All data-holding fields are public, allowing for direct, unchecked modification. The class is a simple data aggregate and contains no internal logic beyond what is required for serialization. It does not cache any data.
-   **Thread Safety:** This class is **not thread-safe**. Its public mutable fields and lack of any synchronization primitives make it inherently unsafe for concurrent access. It is designed to be operated on by a single thread at a time, such as a Netty I/O worker thread.

**WARNING:** Sharing an ItemAnimation instance across threads without external locking will lead to race conditions, memory visibility issues, and unpredictable behavior.

## API Surface
The public contract is dominated by static utility methods for interacting with the binary protocol.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static ItemAnimation | O(N) | Constructs an ItemAnimation by reading from a buffer at a given offset. N is the total size of the variable string data. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided buffer according to the defined binary layout. N is the total size of the variable string data. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a series of checks on a buffer to verify if it contains a valid ItemAnimation structure. Does not perform a full deserialization. |
| computeSize() | int | O(N) | Calculates the total number of bytes this object would occupy when serialized. N is the number of non-null strings. |
| computeBytesConsumed(ByteBuf, int) | static int | O(1) | Reads the header and offset table to calculate the total size of a serialized ItemAnimation structure within a buffer without full deserialization. |

## Integration Patterns

### Standard Usage
The class is intended to be used by protocol handlers that encode or decode network packets. The typical flow involves validating the data structure before attempting a full deserialization to prevent errors and ensure security.

```java
// Deserializing from a network buffer
ByteBuf networkBuffer = ...;
int dataOffset = ...;

ValidationResult result = ItemAnimation.validateStructure(networkBuffer, dataOffset);
if (result.isOk()) {
    ItemAnimation anim = ItemAnimation.deserialize(networkBuffer, dataOffset);
    // Pass the 'anim' object to game logic
    processAnimationData(anim);
} else {
    // Handle invalid packet data
    log.error("Invalid ItemAnimation structure: " + result.getErrorMessage());
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not hold onto instances created by `deserialize` for extended periods. They are meant to be transient. If animation data is needed long-term, copy its values into a more permanent, encapsulated game state object.
-   **Concurrent Modification:** Never modify an ItemAnimation instance from one thread while another thread is reading it or serializing it. This will cause data corruption.
-   **Ignoring Validation:** Skipping the `validateStructure` call before `deserialize` on untrusted input is dangerous. A malformed packet could cause a ProtocolException or other runtime buffer errors, potentially leading to a connection termination.

## Data Pipeline
ItemAnimation serves as a critical translation point between the raw network byte stream and the structured in-memory game state.

> **Inbound Flow:**
> Netty ByteBuf -> Protocol Decoder -> **ItemAnimation.validateStructure** -> **ItemAnimation.deserialize** -> In-Memory ItemAnimation Instance -> Game Logic

> **Outbound Flow:**
> Game Logic -> New ItemAnimation Instance -> **instance.serialize(ByteBuf)** -> Protocol Encoder -> Netty ByteBuf -> Network Socket

