---
description: Architectural reference for AssetInfo
---

# AssetInfo

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class AssetInfo {
```

## Architecture & Concepts

The AssetInfo class is a fundamental Data Transfer Object (DTO) within the Asset Editor's network protocol. It does not represent a service or a manager, but rather the serialized contract for communicating metadata about a single asset between the client and server. Its primary function is to encapsulate the state of an asset—such as its path, modification history, and status (new, deleted, or moved)—into a compact binary format for network transmission.

The design of AssetInfo prioritizes network performance and payload size over ease of use. It employs a custom, non-reflective serialization scheme with a fixed-size header and a variable-size data block. This structure allows for extremely fast parsing and minimal overhead, which is critical in a real-time editing environment.

Key architectural features include:

*   **Fixed/Variable Layout:** The serialized data is split. A fixed-size block of 23 bytes contains booleans, a timestamp, and offsets to variable-length data.
*   **Offset-Based Pointers:** Instead of serializing variable-length fields (like strings or nested objects) inline, the fixed block stores integer offsets pointing to their location in the variable data section of the buffer. This allows for efficient skipping of fields during parsing.
*   **Bitmask for Nullability:** The first byte of the serialized data is a bitmask (`nullBits`) indicating which of the nullable, variable-length fields are present in the payload. This is a common network optimization to avoid wasting bytes on null terminators or length prefixes for absent data.

This class is a foundational element of the protocol layer, acting as the in-memory representation of a network message before it is processed by higher-level application logic.

### Lifecycle & Ownership

-   **Creation:** An AssetInfo object is instantiated under two primary conditions:
    1.  By the network protocol layer when a packet is received, via the static `deserialize` factory method.
    2.  By application logic (e.g., the Asset Editor service) when preparing to send an asset update to a remote peer. In this case, the standard constructor is used.
-   **Scope:** The object is **short-lived** and **transient**. Its lifetime is typically confined to the scope of a single network event or user action. It is created, processed, and then becomes eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency

-   **State:** The AssetInfo class is a **mutable** data container. All of its fields are public and can be directly modified after instantiation. This design is common for DTOs that are constructed incrementally before being serialized.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms.

    **WARNING:** Concurrent access to an AssetInfo instance is inherently unsafe. If an instance must be passed between threads, all access must be externally synchronized, or a defensive copy should be created using the copy constructor or `clone` method. The intended pattern is for a single thread (e.g., a Netty event loop thread) to own and process the object.

## API Surface

The public API is dominated by static methods for serialization, deserialization, and validation, reflecting its role as a data contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetInfo | O(V) | Constructs an AssetInfo object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. V is the number of variable-length fields present. |
| serialize(buf) | void | O(V) | Writes the object's state into the provided ByteBuf according to the custom binary protocol. V is the number of variable-length fields present. |
| validateStructure(buf, offset) | static ValidationResult | O(V) | Performs a safe, read-only check of the buffer to ensure the data represents a valid AssetInfo structure. **CRITICAL:** This must be called on untrusted data before attempting to deserialize. |
| computeSize() | int | O(V) | Calculates the exact number of bytes this object will consume when serialized. Useful for pre-allocating buffers. |
| computeBytesConsumed(buf, offset) | static int | O(V) | Calculates the size of a serialized AssetInfo directly from a buffer without full deserialization. Useful for advancing a buffer cursor past the object. |

## Integration Patterns

### Standard Usage

The most common use case is decoding an object from a network buffer within a protocol handler. The process must involve validation before deserialization.

```java
// In a network packet handler, given a ByteBuf
ByteBuf packetData = ...;
int readOffset = ...;

// 1. Validate the structure before parsing to prevent errors
ValidationResult result = AssetInfo.validateStructure(packetData, readOffset);
if (!result.isValid()) {
    throw new ProtocolException("Invalid AssetInfo structure: " + result.error());
}

// 2. Deserialize the object
AssetInfo info = AssetInfo.deserialize(packetData, readOffset);

// 3. Process the data
assetEditorService.handleAssetUpdate(info);
```

### Anti-Patterns (Do NOT do this)

-   **Ignoring Validation:** Never call `deserialize` on a buffer received from a remote source without first calling `validateStructure`. Bypassing this step exposes the system to potential `ProtocolException` crashes or, in worse cases, buffer over-reads from maliciously crafted packets.
-   **Long-Term State:** Do not retain AssetInfo instances as part of long-term application state. They are designed as transient messages. If the data needs to be preserved, map it to a stable, internal domain model.
-   **Cross-Thread Modification:** Do not modify an AssetInfo instance on one thread while another thread is reading from it or serializing it. This will lead to race conditions and unpredictable data corruption in the serialized payload.

## Data Pipeline

AssetInfo serves as the structured, in-memory representation of asset metadata as it flows from the network into the application logic.

> Flow:
> Raw ByteBuf from Netty -> Protocol Frame Decoder -> **AssetInfo.deserialize** -> In-Memory AssetInfo Object -> Asset Editor Service -> Game State / UI Update

