---
description: Architectural reference for AssetEditorFetchAssetReply
---

# AssetEditorFetchAssetReply

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class AssetEditorFetchAssetReply implements Packet {
```

## Architecture & Concepts
The AssetEditorFetchAssetReply class is a network packet data structure within the Hytale protocol. It serves a single, critical purpose: to transport the raw byte data of a requested asset from a server or asset source to a client-side Asset Editor. This class represents the *response* in an asynchronous request-reply pattern, where a client first sends an AssetEditorFetchAsset request.

It is fundamentally a data container, devoid of business logic beyond what is required for network serialization and deserialization. The design is optimized for performance and strict network buffer layout, characteristic of a protocol that has been defined by a schema and implemented via code generation.

Key architectural components include:
- **Packet ID:** The static field PACKET_ID (312) is a unique identifier used by the protocol dispatcher to route incoming byte streams to the correct deserializer for this packet type.
- **Request Token:** The public field *token* is an integer used to correlate this reply with a specific, prior request. This is essential in asynchronous systems where multiple asset requests may be in-flight simultaneously. The client uses this token to match the incoming asset data to the correct pending operation.
- **Nullable Payload:** The *contents* field is a byte array that can be null. A null value signifies that the requested asset could not be found or retrieved, while a non-null value contains the raw binary data of the asset itself (e.g., a PNG image, JSON model data).

## Lifecycle & Ownership
- **Creation:** An instance is created under two primary scenarios:
    1.  **On Receive:** The network layer's packet dispatcher invokes the static `deserialize` method when an incoming network buffer with packet ID 312 is detected. This is the most common creation path on the client.
    2.  **On Send:** Server-side logic instantiates this class via its constructor (`new AssetEditorFetchAssetReply(...)`) when preparing a response to an asset fetch request. The server populates the token and asset contents before passing the object to the serializer.
- **Scope:** The object's lifetime is extremely brief and transient. It exists only for the duration of its journey through the packet handling pipeline. Once its data (the *token* and *contents*) has been extracted and passed to the appropriate system, the object is no longer referenced and becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or manual cleanup steps required.

## Internal State & Concurrency
- **State:** The class is a mutable data holder. Its public fields, *token* and *contents*, can be modified after instantiation. The state consists solely of the data being transported and contains no derived or cached information.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, processed, and discarded within the context of a single thread, typically a Netty I/O worker or a dedicated game logic thread. Sharing an instance across multiple threads without external synchronization mechanisms is an error and will lead to unpredictable behavior. The `clone` method is provided to create a deep copy for safe data handoff between threads if absolutely necessary.

## API Surface
The public API is focused on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static AssetEditorFetchAssetReply | O(N) | Constructs a new instance by reading from a Netty ByteBuf. N is the size of the asset contents. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the instance's state into a Netty ByteBuf for network transmission. N is the size of the asset contents. |
| computeSize() | int | O(1) | Calculates the exact number of bytes this packet will occupy on the wire. This is a fast metadata operation. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a lightweight check on a buffer to see if it contains a structurally valid packet of this type without full deserialization. |
| clone() | AssetEditorFetchAssetReply | O(N) | Creates a deep copy of the object, including a new copy of the contents byte array. |
| getId() | int | O(1) | Returns the static packet ID (312). |

## Integration Patterns

### Standard Usage
This packet is almost never instantiated or manipulated directly by feature developers. Instead, it is processed by a registered packet handler. The handler extracts the data and completes a future or promise associated with the original request.

```java
// Example of a packet handler processing the reply
public class AssetEditorPacketHandler {

    // A map holding promises for pending asset requests, keyed by token
    private final Map<Integer, Promise<byte[]>> pendingRequests;

    public void handle(AssetEditorFetchAssetReply reply) {
        Promise<byte[]> promise = pendingRequests.remove(reply.token);
        if (promise != null) {
            // Fulfill the promise with the asset contents (which can be null)
            promise.complete(reply.contents);
        } else {
            // Log a warning: received a reply for an unknown or timed-out request
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Deserialization:** Do not use `new AssetEditorFetchAssetReply()` and manually read from a ByteBuf to populate its fields. This bypasses critical validation and length checks, leading to potential buffer overflows or `IndexOutOfBoundsException`. Always use the static `deserialize` method.
- **Ignoring the Token:** The *token* field is the only mechanism to correlate the reply with the request. Discarding it or using a default value will break the asset loading system.
- **State Reuse:** Do not modify and re-send an existing packet instance. Packet objects should be treated as immutable after creation for sending. Create a new instance for each distinct message.
- **Cross-Thread Sharing:** Do not pass an instance from the network thread to a worker thread without cloning it or ensuring a proper memory visibility barrier.

## Data Pipeline
The primary flow for this packet is from the server's asset system to the client's asset system. The following illustrates the data pipeline upon receipt by the client.

> Flow:
> Network ByteBuf -> Protocol Dispatcher -> **AssetEditorFetchAssetReply.deserialize** -> Packet Handler -> Asset Request Promise -> Asset Editor UI / Cache

