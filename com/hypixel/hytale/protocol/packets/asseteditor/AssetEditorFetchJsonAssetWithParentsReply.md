---
description: Architectural reference for AssetEditorFetchJsonAssetWithParentsReply
---

# AssetEditorFetchJsonAssetWithParentsReply

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Object

## Definition
```java
// Signature
public class AssetEditorFetchJsonAssetWithParentsReply implements Packet {
```

## Architecture & Concepts
The AssetEditorFetchJsonAssetWithParentsReply class is a network packet data structure, a fundamental component of the client-server communication protocol. It serves as a server-to-client message, specifically designed for the in-game asset editor tooling. Its primary function is to transport the string content of a requested JSON asset, along with the content of all its parent assets, back to the client that initiated the request.

This packet is part of a request-reply pattern. A client-side tool sends a corresponding request packet (likely AssetEditorFetchJsonAssetWithParents) containing a unique token. The server processes this request, gathers the required asset data, and encapsulates it within this reply packet, including the original token for correlation.

The class design is self-contained, bundling serialization, deserialization, size computation, and validation logic. This adheres to a common network protocol pattern where each packet is an autonomous unit, simplifying the work of the higher-level protocol manager which dispatches raw byte buffers to the correct packet implementation based on a packet identifier.

## Lifecycle & Ownership
- **Creation:**
    - **Server-Side:** Instantiated by the server's asset management service in response to a client request. The constructor is called with the request token and a map of the resolved asset data.
    - **Client-Side:** Instantiated exclusively by the protocol layer's deserialization logic when a network message with Packet ID 313 is received. The static `deserialize` method is the entry point for its creation on the client.

- **Scope:** This object has an extremely brief, ephemeral lifecycle.
    - On the server, it exists only for the duration of the serialization process before being written to the network buffer.
    - On the client, it exists from the moment of deserialization until the relevant packet handler has consumed its data.

- **Destruction:** The object is eligible for garbage collection immediately after its payload (the `assets` map) has been processed by the client's application logic. There are no external references maintained by the protocol framework after the handler completes.

## Internal State & Concurrency
- **State:** The object's state is mutable. Its public fields, `token` and `assets`, can be directly modified. This design facilitates the step-by-step construction of the object during the deserialization process. Once deserialization is complete and the packet is passed to a handler, its state should be treated as immutable.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, processed, and discarded within a single network thread (e.g., a Netty event loop). Any concurrent access or modification from multiple threads requires external synchronization and is strongly discouraged, as it violates the assumptions of the network protocol layer.

## API Surface
The public API is dominated by static methods for network buffer processing and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static AssetEditorFetchJsonAssetWithParentsReply | O(N) | Constructs a new instance by reading from a network buffer. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the instance's state into a network buffer. Throws ProtocolException if constraints are violated. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the size of a serialized object directly from a buffer without full deserialization. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a lightweight check on a buffer to ensure it contains a structurally valid packet of this type. |
| getId() | int | O(1) | Returns the static packet identifier, 313. |

*N represents the total size of the asset data in the `assets` map.*

## Integration Patterns

### Standard Usage
This packet is not intended for direct use by most game logic developers. It is handled by the client's asset editor subsystem. A handler receives the packet, correlates the token with a pending request, and uses the asset map to update its internal state or UI.

```java
// Example of a client-side packet handler
public void handleAssetReply(AssetEditorFetchJsonAssetWithParentsReply reply) {
    // Correlate the token with an outstanding request
    PendingAssetRequest pending = assetRequestManager.getPending(reply.token);
    if (pending != null) {
        // Fulfill the request with the received data
        pending.complete(reply.assets);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Retention:** Do not store instances of this packet. Its data should be immediately extracted into application-specific data structures. Holding onto the packet object prevents it from being garbage collected and serves no purpose.
- **State Mutation:** Do not modify the packet's fields after it has been received by a handler. The object should be treated as a read-only data transfer object post-deserialization.
- **Client-Side Instantiation:** Never use `new AssetEditorFetchJsonAssetWithParentsReply()` on the client. The only valid way to create this object on the client is through the protocol layer's deserialization process.

## Data Pipeline
The flow of this data object is unidirectional from server to client as part of a larger request-response cycle.

> **Server Flow:**
> Client Request -> Asset Service Logic -> **AssetEditorFetchJsonAssetWithParentsReply (Instantiation)** -> Protocol Serializer -> Netty ByteBuf -> Network
>
> **Client Flow:**
> Network -> Netty ByteBuf -> Protocol Deserializer -> **AssetEditorFetchJsonAssetWithParentsReply (Instantiation)** -> Packet Handler -> Asset Editor Cache/UI

