---
description: Architectural reference for AssetEditorAssetType
---

# AssetEditorAssetType

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class AssetEditorAssetType {
```

## Architecture & Concepts
The AssetEditorAssetType class is a Data Transfer Object (DTO) that serves as a strongly-typed, in-memory representation of a network packet payload. Its primary function is to serialize and deserialize data describing a specific asset type used within the Hytale Asset Editor.

This class is a critical component of the network protocol layer, acting as the translation boundary between the raw binary stream on the wire and the structured objects used by the game engine's logic. It is not a service or manager; it is a pure data container with no inherent behavior beyond data marshalling.

The underlying binary format is highly optimized for network efficiency. It employs a fixed-size block followed by a variable-size block.
1.  **Fixed Block:** Contains a bitmask (`nullBits`) indicating which of the nullable string fields are present, followed by fixed-size data like booleans and enums. It also contains integer offsets pointing to the location of each variable-length string.
2.  **Variable Block:** Contains the actual UTF-8 encoded string data, packed contiguously.

This structure allows for extremely fast initial parsing and validation, as a reader can process the fixed-size header to understand the full payload's structure and size without needing to parse the variable-length strings immediately.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary scenarios:
    1.  **Inbound:** The static factory method `deserialize` is called by a network protocol decoder (e.g., a Netty channel handler) when a corresponding packet arrives from a client or server.
    2.  **Outbound:** Game logic instantiates it directly using its constructor when preparing to send asset information over the network.
- **Scope:** The object's lifetime is intentionally brief. It is scoped to the processing of a single network message. It is created, its data is consumed by a handler, and it is then discarded.
- **Destruction:** Managed entirely by the Java Garbage Collector. Once all references to the instance are released, typically after a network event handler completes its execution, the object becomes eligible for collection. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The object's state is fully mutable via its public fields. It is designed as a simple data container to be populated once (either via deserialization or construction) and then read by game logic. It performs no internal caching.
- **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single thread, such as a Netty I/O worker thread or the main game thread. Modifying its fields from multiple threads without external synchronization will result in data corruption and undefined behavior. All serialization and deserialization operations on a given `ByteBuf` must be confined to the thread that owns the buffer.

## API Surface
The public contract is dominated by static methods for protocol handling and the instance method for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | AssetEditorAssetType | O(N) | **[Factory]** Deserializes an object from a `ByteBuf` at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Serializes the object's state into the provided `ByteBuf`. |
| validateStructure(buf, offset) | ValidationResult | O(1) | Performs critical bounds and length checks on the raw buffer before deserialization. Does not parse full strings. |
| computeBytesConsumed(buf, offset) | int | O(1) | Calculates the total size of the serialized object in a buffer by reading its structural metadata. |
| computeSize() | int | O(N) | Calculates the required buffer size to serialize the current object state. |

*Complexity O(N) refers to the total length of the string data within the object.*

## Integration Patterns

### Standard Usage
The class should be used as part of a network decoding pipeline. Validation is a mandatory first step before attempting to deserialize untrusted data.

```java
// Example from within a Netty ChannelInboundHandler or similar decoder

// 1. Validate the structure before proceeding
ValidationResult result = AssetEditorAssetType.validateStructure(networkBuffer, readOffset);
if (result.isError()) {
    // Disconnect the client or log a protocol violation.
    // DO NOT attempt to deserialize.
    throw new ProtocolException("Invalid AssetEditorAssetType structure: " + result.getErrorMessage());
}

// 2. Deserialize the object
AssetEditorAssetType assetType = AssetEditorAssetType.deserialize(networkBuffer, readOffset);

// 3. Pass the safe, structured object to game logic
gameLogic.handleAssetTypeUpdate(assetType);

// 4. Advance the buffer reader
int bytesConsumed = AssetEditorAssetType.computeBytesConsumed(networkBuffer, readOffset);
networkBuffer.readerIndex(readOffset + bytesConsumed);
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Validation:** Never call `deserialize` on a buffer received from the network without first calling `validateStructure`. Bypassing this check exposes the server to denial-of-service attacks via malformed packets that specify excessively large string lengths, potentially leading to `OutOfMemoryError`.
- **Instance Re-use:** Do not attempt to pool or re-use `AssetEditorAssetType` instances for different messages. They are lightweight objects, and re-using them can lead to subtle state corruption bugs. Create a new instance for each message being sent or received.
- **Cross-thread Sharing:** Do not pass an instance to another thread for modification without proper synchronization. If data needs to be passed to another thread, it is safer to create a new, immutable copy or use a thread-safe data structure.

## Data Pipeline

The class serves as the translation point in the data pipeline between the network layer and the application layer.

> **Inbound Flow:**
> Raw TCP Stream -> Netty ByteBuf -> **AssetEditorAssetType.validateStructure** -> **AssetEditorAssetType.deserialize** -> Game Logic Handler

> **Outbound Flow:**
> Game Logic Event -> `new AssetEditorAssetType(...)` -> **AssetEditorAssetType.serialize** -> Netty ByteBuf -> Raw TCP Stream

