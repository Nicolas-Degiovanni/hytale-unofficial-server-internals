---
description: Architectural reference for AssetEditorUndoRedoReply
---

# AssetEditorUndoRedoReply

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorUndoRedoReply implements Packet {
```

## Architecture & Concepts
The AssetEditorUndoRedoReply is a network Data Transfer Object (DTO) that operates within the Hytale Protocol Layer. It serves as a server-to-client message, specifically designed to confirm the result of an undo or redo operation initiated by a client within the in-game asset editor.

This packet is the response half of a request-reply pattern. The client sends an AssetEditorUndoRedoRequest, and the server processes it and returns this AssetEditorUndoRedoReply. The core architectural components are:

*   **Token Correlation:** The public `token` field is a critical identifier. It allows the client to match this incoming reply with the specific request it sent, preventing state corruption if multiple requests are in flight.
*   **Payload Delivery:** The nullable `command` field, of type JsonUpdateCommand, carries the substantive data. It represents the state change that was reverted (undo) or reapplied (redo). If an operation fails or results in no state change, this field may be null.
*   **Binary Protocol:** This class implements a highly optimized, custom serialization and deserialization scheme. It uses a bitmask (`nullBits`) to efficiently encode the presence of nullable fields and defines a fixed block size for predictable parsing by the network layer.

This class has no business logic. Its sole responsibility is to represent and transport data across the network boundary in a structured and efficient binary format.

### Lifecycle & Ownership
- **Creation:** An instance is created in one of two scenarios:
    1.  **On the Server:** The asset editor's backend logic instantiates this class to formulate a reply to a client's request.
    2.  **On the Client:** The static `deserialize` method is invoked by a Netty pipeline handler when a raw network buffer containing a packet with ID 351 is received.
- **Scope:** This object is extremely short-lived and has a narrow scope. It is designed to exist only for the duration of processing within a single network event or game tick. It is not intended to be stored or referenced beyond the handler that processes it.
- **Destruction:** The object becomes eligible for garbage collection as soon as the network event handler or game logic block that received it completes execution. There are no engine systems that hold long-term references to packet instances.

## Internal State & Concurrency
- **State:** The object's state is mutable through its public fields. However, by convention, it should be treated as immutable after deserialization on the client or before serialization on the server. Its purpose is to be a snapshot of a command at a point in time.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, serialized, deserialized, and processed on a single thread, typically a Netty I/O worker or the main game thread. Passing this object between threads requires explicit external synchronization, which is a significant anti-pattern.

## API Surface
The primary contract is for serialization and deserialization, used exclusively by the protocol layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetEditorUndoRedoReply | O(N) | **Factory Method.** Constructs a new instance by reading from a Netty ByteBuf. N is the size of the nested command. |
| serialize(buf) | void | O(N) | Encodes the object's state into a binary representation and writes it to the provided ByteBuf. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will consume when serialized. Used for buffer pre-allocation. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a low-cost structural integrity check on a buffer without full deserialization. Fails if data is malformed. |

## Integration Patterns

### Standard Usage
The object is almost exclusively handled by the network layer and the client-side systems responsible for managing asset editor state. A developer would typically interact with it inside a packet handler.

```java
// Hypothetical client-side packet handler
public void handleUndoRedoReply(AssetEditorUndoRedoReply reply) {
    // Retrieve the original request context using the token
    PendingRequest pending = this.requestTracker.get(reply.token);
    if (pending == null) {
        // Reply for an unknown or timed-out request; discard
        return;
    }

    // If a command was returned, apply it to the local asset state
    if (reply.command != null) {
        this.assetStateManager.applyCommand(reply.command);
    }

    // Update UI to reflect the completed operation
    this.uiController.showSuccessToast("Operation successful!");
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not cache or store instances of this packet. They represent a transient event, not persistent state. Caching them can lead to memory leaks and use of stale data.
- **State Mutation:** Do not modify the `token` or `command` fields after the packet has been deserialized. Treat it as a read-only record of a network event.
- **Cross-Thread Access:** Never share an instance of this packet across threads without proper locking. The internal state is not protected, and doing so will lead to unpredictable behavior and data corruption.

## Data Pipeline
The data flow for this packet is unidirectional from the server to the client as a response to a prior action.

> Flow:
> Server Asset Logic -> **AssetEditorUndoRedoReply (Instance Creation & Serialization)** -> Network Encoder -> TCP/IP -> Client Network Decoder -> **AssetEditorUndoRedoReply (Deserialization)** -> Client Packet Handler -> Asset State Manager -> UI Update

