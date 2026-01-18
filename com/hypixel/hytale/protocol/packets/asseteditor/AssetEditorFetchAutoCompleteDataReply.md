---
description: Architectural reference for AssetEditorFetchAutoCompleteDataReply
---

# AssetEditorFetchAutoCompleteDataReply

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class AssetEditorFetchAutoCompleteDataReply implements Packet {
```

## Architecture & Concepts
The AssetEditorFetchAutoCompleteDataReply class is a network packet definition within the Hytale protocol layer. It serves a single, specific purpose: to transmit auto-completion suggestions from the server to the client for use within the in-game Asset Editor.

This class is a pure data container, representing the *reply* in a request-reply asynchronous communication pattern. The client initiates a request for auto-complete data (likely an AssetEditorFetchAutoCompleteData packet), including a unique `token`. The server processes this request and responds with this reply packet, embedding the same `token`. This token is critical for the client-side Asset Editor to correlate the incoming results with the correct UI element or context that made the original request, preventing race conditions and data mismatches in the user interface.

As an implementation of the Packet interface, this class is a fundamental component of the network serialization and deserialization pipeline. The protocol engine relies on its static methods like `deserialize` and `validateStructure` to safely construct the object from a raw byte stream received from the network.

## Lifecycle & Ownership
- **Creation:**
    - **Server-Side:** Instantiated by a server-side packet handler when it has generated the list of auto-complete strings in response to a client's request. The handler populates the `token` and `results` fields before passing the object to the network layer for serialization.
    - **Client-Side:** Instantiated by the protocol's deserialization engine. When a network buffer with the packet ID 332 is received, the static `deserialize` method is invoked to construct the object from the byte stream. It is never created using its constructor on the client.

- **Scope:** This object has an extremely short and transient lifecycle.
    - On the server, it exists only for the duration of the serialization process before being sent over the network.
    - On the client, it exists from the moment of deserialization until it is consumed by the appropriate packet handler, which extracts its data to update the Asset Editor UI. It is then immediately eligible for garbage collection.

- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or manual cleanup procedures required.

## Internal State & Concurrency
- **State:** This class is a mutable data container. Its fields, `token` and `results`, are public and can be modified after instantiation. However, by design, it is treated as an immutable value object after it has been populated (on the server) or deserialized (on the client).

- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization mechanisms.
    - **Warning:** It is designed to be created, serialized, deserialized, and processed by a single thread at each stage of its lifecycle. On the client, this typically means it is deserialized on a Netty network thread and then passed to the main game thread via a queue for safe processing. Direct access from multiple threads will result in undefined behavior.

## API Surface
The public API is dominated by static methods used by the protocol engine for serialization and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetEditorFetchAutoCompleteDataReply | O(N) | Constructs an object from a ByteBuf. Throws ProtocolException on malformed data. N is the total size of the string data. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf. Throws ProtocolException if data constraints are violated. |
| computeSize() | int | O(N) | Calculates the number of bytes required to serialize the object. Critical for buffer pre-allocation. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a read-only check of the buffer to ensure it contains a valid packet structure without full deserialization. |
| clone() | AssetEditorFetchAutoCompleteDataReply | O(M) | Creates a shallow copy of the object, with a deep copy of the results array. M is the number of strings. |

## Integration Patterns

### Standard Usage
The class is used exclusively by the network protocol layer and the Asset Editor's business logic. A developer would typically interact with it inside a packet handler.

```java
// Example of a client-side packet handler
public void handle(AssetEditorFetchAutoCompleteDataReply reply) {
    // Retrieve the UI component associated with the original request token
    AutoCompleteUIComponent ui = findUiComponentByToken(reply.token);

    if (ui != null) {
        // Safely update the UI with the results from the main game thread
        ui.populateSuggestions(reply.results);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation on Client:** Do not use `new AssetEditorFetchAutoCompleteDataReply()` on the client. Packets are created by the deserialization pipeline.
- **Reusing Instances:** Do not hold references to packet instances or attempt to reuse them. They are designed to be single-use, transient objects.
- **Ignoring the Token:** The `token` field is the only mechanism to prevent race conditions between multiple concurrent auto-complete requests. Ignoring it will lead to corrupted UI state.
- **Manual Serialization:** Application-level code should never call `serialize` or `deserialize`. This is the sole responsibility of the protocol engine and its Netty pipeline handlers.

## Data Pipeline
The flow of data encapsulated by this packet is unidirectional from server to client.

> **Server Flow:**
> Asset Editor Service (generates results) -> Creates **AssetEditorFetchAutoCompleteDataReply** instance -> Protocol Serializer -> Netty ByteBuf -> Network Socket

> **Client Flow:**
> Network Socket -> Netty ByteBuf -> Protocol Deserializer -> Creates **AssetEditorFetchAutoCompleteDataReply** instance -> Packet Handler -> Asset Editor UI Logic

