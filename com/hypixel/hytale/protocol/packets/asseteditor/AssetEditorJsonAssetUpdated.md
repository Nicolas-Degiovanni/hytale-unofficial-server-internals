---
description: Architectural reference for AssetEditorJsonAssetUpdated
---

# AssetEditorJsonAssetUpdated

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorJsonAssetUpdated implements Packet {
```

## Architecture & Concepts

The AssetEditorJsonAssetUpdated class is a Data Transfer Object (DTO) that represents a network message within Hytale's proprietary protocol. It is not a service or manager, but rather a structured container for data in transit. Its primary function is to communicate a set of incremental changes to a specific JSON-based game asset from a source (likely a development tool or asset editor) to a destination (a game client or server).

This packet is a critical component of the live asset editing pipeline. Its design emphasizes network efficiency and robustness. Instead of transmitting the entire modified JSON file, it sends an array of JsonUpdateCommand objects. This "patch" or "diff" based approach significantly reduces payload size, which is essential for maintaining a responsive real-time editing experience.

The serialization and deserialization logic is manually implemented against a Netty ByteBuf, indicating a highly optimized, custom binary protocol. This avoids the overhead of text-based formats and provides granular control over the byte-level layout of the data.

### Lifecycle & Ownership
- **Creation:**
    - On the **sending system** (e.g., an asset editor), an instance is created via its constructor, populated with the asset path and a list of commands representing user changes.
    - On the **receiving system** (e.g., a game client), an instance is created exclusively by the static `deserialize` method, which decodes an incoming network buffer.
- **Scope:** The object's lifetime is extremely short and scoped to a single network event. It is created, processed by a handler, and then immediately becomes eligible for garbage collection. It is not designed to be stored or referenced long-term.
- **Destruction:** There is no explicit destruction method. The Java Garbage Collector reclaims the object's memory once all references to it are dropped, which typically occurs upon exiting the network packet handler method.

## Internal State & Concurrency
- **State:** The class is a mutable data container. Its public fields, `path` and `commands`, can be modified after instantiation. However, by convention, it should be treated as immutable after being populated on the sending side or after being deserialized on the receiving side.
- **Thread Safety:** This class is **not thread-safe**. It is a plain data object with no internal locking mechanisms. All operations, including creation, serialization, deserialization, and reading, are expected to occur within a single thread, typically a Netty I/O worker thread. Passing this object between threads requires external synchronization, which is a significant anti-pattern.

## API Surface
The public contract is defined by the Packet interface and the static utility methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetEditorJsonAssetUpdated | O(N) | Constructs an object by reading from a binary buffer. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into a binary buffer for network transmission. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a pre-check of the buffer to validate offsets and lengths without full deserialization. Critical for security. |
| computeSize() | int | O(N) | Calculates the required buffer size for serialization. Used for buffer pre-allocation. |
| getId() | int | O(1) | Returns the static packet identifier (325). |

*N represents the number of commands in the payload.*

## Integration Patterns

### Standard Usage
This packet is processed within a network pipeline. A handler receives a raw buffer, identifies the packet ID, and delegates to the static `deserialize` method. The resulting object is then passed to a higher-level service for processing.

```java
// Example within a hypothetical packet handler
void handlePacket(ByteBuf buffer) {
    // A dispatcher would have already identified the packet ID as 325
    
    // First, validate the packet structure to prevent parsing errors or attacks
    ValidationResult result = AssetEditorJsonAssetUpdated.validateStructure(buffer, buffer.readerIndex());
    if (!result.isValid()) {
        throw new ProtocolException("Invalid AssetEditorJsonAssetUpdated packet: " + result.error());
    }

    // If valid, deserialize into a usable object
    AssetEditorJsonAssetUpdated updatePacket = AssetEditorJsonAssetUpdated.deserialize(buffer, buffer.readerIndex());

    // Pass the structured data to the responsible system
    AssetService assetService = context.getService(AssetService.class);
    assetService.applyJsonUpdate(updatePacket.path, updatePacket.commands);
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Deserialization:** Do not attempt to read the fields from the ByteBuf manually. Always use the provided static `deserialize` method to ensure correctness.
- **Ignoring Validation:** Never deserialize a packet from an untrusted source without first calling `validateStructure`. Skipping this step can expose the application to denial-of-service attacks via maliciously crafted packets (e.g., an invalid array length causing an OutOfMemoryError).
- **Object Reuse:** Do not modify and re-serialize a deserialized packet object. Treat them as single-use, immutable data records once received.
- **Cross-Thread Access:** Do not share instances of this packet across threads without proper and explicit synchronization. The object is not designed for concurrent access.

## Data Pipeline
The AssetEditorJsonAssetUpdated packet is a key link in the data flow for real-time asset modification.

> Flow:
> User Action in Editor -> Change Commands Generated -> **AssetEditorJsonAssetUpdated.serialize()** -> Netty Channel -> Network -> Packet Framer -> **AssetEditorJsonAssetUpdated.deserialize()** -> Packet Handler -> Asset Management Service -> Live Game Asset Patched

