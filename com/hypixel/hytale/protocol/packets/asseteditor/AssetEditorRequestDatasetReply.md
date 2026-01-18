---
description: Architectural reference for AssetEditorRequestDatasetReply
---

# AssetEditorRequestDatasetReply

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorRequestDatasetReply implements Packet {
```

## Architecture & Concepts

The AssetEditorRequestDatasetReply class is a Data Transfer Object (DTO) that operates exclusively within the Hytale network protocol layer. Its primary function is to transport a named dataset, consisting of a collection of string identifiers, from the server to the client. This packet is a direct response to a client-side request initiated within the Asset Editor.

Architecturally, this class represents a single, immutable message format. It is not a service or a manager; it is a data container with a highly specialized binary serialization contract. The serialization format is optimized for performance, using a null-bit field and relative offsets to handle variable-length data efficiently. This design minimizes payload size and parsing overhead on the network thread.

This packet is fundamental to the interactive nature of the in-game content creation tools, allowing the client to dynamically query the server for available assets (e.g., model names, texture paths) without needing a complete asset manifest upfront.

### Lifecycle & Ownership
-   **Creation:**
    -   On the server, an instance is created and populated by a request handler responsible for querying asset information.
    -   On the client, an instance is created exclusively by the network protocol layer's `deserialize` method when a packet with ID 334 is received.
-   **Scope:** The object's lifetime is extremely short and bound to the scope of a single network event processing cycle. On the server, it exists only long enough to be serialized into a Netty ByteBuf. On the client, it exists only until its data is consumed by a packet handler and transferred to a more persistent application state object.
-   **Destruction:** The object is eligible for garbage collection immediately after its data has been processed. There are no external references maintained by the framework.

## Internal State & Concurrency
-   **State:** The class holds mutable state via its public fields, `name` and `ids`. However, it is intended to be treated as immutable after its initial population (on the server) or deserialization (on the client). The internal data represents a snapshot of asset information at a specific moment.
-   **Thread Safety:** This class is **not thread-safe**. Its public, mutable fields make it inherently unsafe for concurrent modification. It is designed to be created, processed, and discarded by a single thread, typically a Netty I/O worker thread.

**WARNING:** Never share an instance of this packet across multiple threads. If data must be passed to another thread (e.g., the main game thread), extract the primitive data (the String and String array) and pass those instead.

## API Surface

The public contract is defined by the Packet interface and the static serialization methods. The fields are public for high-performance access by the protocol codec.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | AssetEditorRequestDatasetReply | O(N) | **[Critical]** Constructs a new packet by parsing a binary representation from a ByteBuf. N is the total size of the string data. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | **[Critical]** Encodes the object's state into a binary representation in the provided ByteBuf. N is the total size of the string data. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a read-only check of the binary data to ensure structural integrity without full deserialization. Used for security and diagnostics. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. |

## Integration Patterns

### Standard Usage

This packet is never directly instantiated or managed by feature developers. It is an implementation detail of the network layer. A client-side system would register a handler for this packet type, which is then invoked by the network dispatcher.

```java
// Hypothetical client-side packet handler
public class AssetEditorPacketHandler {

    private final AssetEditorUI assetEditorUI;

    public void handle(AssetEditorRequestDatasetReply reply) {
        // This method is invoked on a network thread.
        // The data is extracted and passed to the main thread for UI updates.
        String datasetName = reply.name;
        String[] assetIds = reply.ids;

        // Schedule a task to run on the main game thread
        GameLoop.getInstance().submitTask(() -> {
            assetEditorUI.populateDataset(datasetName, assetIds);
        });
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new AssetEditorRequestDatasetReply()` on the client. Packets are created by the deserialization process.
-   **State Mutation:** Do not modify the public fields of a received packet. This can lead to unpredictable behavior if other handlers expect the original data.
-   **Long-Term Storage:** Do not hold references to this packet object. Copy its data into application-level state containers and allow the packet to be garbage collected.

## Data Pipeline

The flow of this data object is linear and unidirectional from the server's data source to the client's user interface.

> Flow:
> Server Asset Database -> Server Request Handler -> **AssetEditorRequestDatasetReply (instantiated & serialized)** -> Netty TCP Channel -> Client Network Layer -> **AssetEditorRequestDatasetReply (deserialized)** -> Client Packet Handler -> Game Thread Scheduler -> Asset Editor UI State Update

