---
description: Architectural reference for AssetEditorFetchAsset
---

# AssetEditorFetchAsset

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class AssetEditorFetchAsset implements Packet {
```

## Architecture & Concepts
AssetEditorFetchAsset is a network packet that represents a command from a client to a server, specifically within the live asset editing subsystem. Its primary role is to request the raw data of a specific game asset, identified by an AssetPath. This packet is a fundamental component of the request-response pattern used by the external asset editor to communicate with a running game instance.

When the asset editor needs to display or allow modification of an asset, it constructs and sends this packet. The receiving end, typically a game client or server, processes this request, retrieves the asset from its storage or memory, and sends the data back in a corresponding response packet. The *token* field is crucial for this asynchronous workflow, allowing the editor to correlate the incoming asset data with the original request it sent.

This class is not a service or manager; it is inert data. Its logic is confined to serialization and deserialization for network transport.

### Lifecycle & Ownership
- **Creation:**
    - **Sending Client (Asset Editor):** Instantiated directly when a user action requires fetching an asset from the game. For example, `new AssetEditorFetchAsset(token, path, true)`.
    - **Receiving Peer (Game Instance):** Instantiated by the network protocol layer when a `ByteBuf` with Packet ID 310 is received. The static `deserialize` method is called to construct the object from the network buffer.
- **Scope:** Transient and extremely short-lived. An instance exists only for the duration of a single network transaction. On the sender side, it is created, serialized, and then becomes eligible for garbage collection. On the receiver side, it is created via deserialization, processed by a single packet handler, and then immediately becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or `close` methods required.

## Internal State & Concurrency
- **State:** Mutable. This object is a simple data container with public fields. Its state is defined by the `token`, `path`, and `isFromOpenedTab` values. The mutability is a design choice for performance, allowing the static `deserialize` method to populate a newly created instance without the overhead of a builder pattern.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, serialized, and processed within a single thread, typically a Netty I/O worker thread.
    - **WARNING:** Sharing an instance of AssetEditorFetchAsset across multiple threads without explicit, external synchronization will lead to race conditions and unpredictable behavior. Do not store instances of this packet in shared collections or pass them between thread contexts.

## API Surface
The public contract of this class is primarily for the network protocol engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | AssetEditorFetchAsset | O(N) | **Static Factory.** Constructs an object from a binary representation in a ByteBuf. N is the size of the AssetPath. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf for network transmission. N is the size of the AssetPath. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this packet will occupy on the wire. N is the size of the AssetPath. |
| validateStructure(buf, offset) | ValidationResult | O(N) | **Static.** Performs a low-level check on a ByteBuf to ensure it contains a structurally valid packet before full deserialization. |

## Integration Patterns

### Standard Usage
This packet is handled by the network layer. A developer would typically interact with it inside a packet handler or when dispatching a command to the asset editor service.

```java
// Example: Sending a request from the asset editor
int transactionToken = generateToken();
AssetPath assetToFetch = new AssetPath("hytale:models/creatures/creeper.json");
AssetEditorFetchAsset request = new AssetEditorFetchAsset(transactionToken, assetToFetch, false);

// The network client serializes and sends the packet
networkClient.send(request);
```

```java
// Example: Receiving and processing the packet in the game
// This logic resides within a network packet handler.
public void handle(AssetEditorFetchAsset packet) {
    // Use the token to track the request
    int token = packet.token;
    AssetPath path = packet.path;

    // Asynchronously load the asset and send it back
    Asset asset = assetService.load(path);
    AssetEditorAssetData response = new AssetEditorAssetData(token, asset.getRawBytes());
    networkClient.send(response);
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Send:** Do not modify the packet's fields after it has been passed to the network layer for sending. The serialization may happen on a different thread, leading to a data race.
- **Reusing Instances:** Do not reuse packet instances for multiple requests. Always create a new object for each distinct network message to ensure state integrity.
- **Cross-Thread Sharing:** Never share a single instance of this packet between threads. It is not designed for concurrent access.

## Data Pipeline
The flow of this data object is unidirectional from the asset editor to the game instance.

> Flow:
> Asset Editor User Action -> **new AssetEditorFetchAsset()** -> Serialization -> Netty I/O Thread -> Network Socket -> Game Instance Network Layer -> Deserialization -> Packet Handler -> Asset Service

