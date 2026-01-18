---
description: Architectural reference for AssetEditorPopupNotification
---

# AssetEditorPopupNotification

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class AssetEditorPopupNotification implements Packet {
```

## Architecture & Concepts
The **AssetEditorPopupNotification** is a specialized network packet that functions as a Data Transfer Object (DTO). Its sole purpose is to transmit notification events from the server to a client specifically engaged in an Asset Editor session. It is a fundamental component of the real-time feedback loop for content creators, enabling the server to communicate status updates, validation results, or errors directly to the user's interface.

This class is part of the core Hytale protocol layer, designed for high-performance serialization and deserialization over the network. It encapsulates a notification *type* (e.g., Info, Warning, Error) and an optional, localizable **FormattedMessage**. The serialization format is highly optimized, using a bitmask for nullable fields to minimize payload size.

This packet is strictly directional: it is sent from the server to the client. The client's network layer is responsible for deserializing the incoming byte stream into an **AssetEditorPopupNotification** object and dispatching it to the appropriate handler, which then updates the Asset Editor UI.

## Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1. **Server-Side (Sending):** Instantiated directly via its constructor, `new AssetEditorPopupNotification(type, message)`, by server-side logic that needs to send a notification to the client's asset editor.
    2. **Client-Side (Receiving):** Instantiated by the static `deserialize` factory method. This method is invoked internally by the client's network protocol dispatcher when a packet with ID 337 is received from the network stream.
- **Scope:** The object's lifetime is extremely brief and transactional. It exists only long enough to be serialized into a network buffer on the server, or from the moment of deserialization until it is consumed by a UI event handler on the client. It is not designed to be stored or referenced long-term.
- **Destruction:** The object is eligible for garbage collection immediately after its data has been processed. On the server, this is after serialization. On the client, this is after the UI handler has extracted the necessary information to display the popup.

## Internal State & Concurrency
- **State:** The internal state is mutable. The public fields `type` and `message` can be modified after construction. This design facilitates the building of the packet object before it is passed to the network layer for serialization.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal locking or synchronization mechanisms. It is expected to be created, populated, and passed to the network layer from a single thread. On the receiving end, it should be processed by a single thread (e.g., a network thread that dispatches it to the main UI thread) to prevent data races.

**WARNING:** Concurrent modification of an **AssetEditorPopupNotification** instance during serialization will lead to a corrupted network packet and unpredictable client-side behavior.

## API Surface
The public API is dominated by methods related to the **Packet** interface, governing its transformation to and from a network byte stream.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(N) | Serializes the object's state into the provided Netty ByteBuf. N is the size of the message. |
| deserialize(ByteBuf, int) | static AssetEditorPopupNotification | O(N) | Deserializes a new instance from a ByteBuf at a given offset. Throws if buffer is malformed. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this packet will occupy on the network. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a structural validation of the packet data within a buffer without full deserialization. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (337). |

## Integration Patterns

### Standard Usage
This packet is not used directly by most game logic. Instead, a client-side system registers a handler for it. The network layer invokes the handler upon receipt.

```java
// Hypothetical client-side packet handler
public class AssetEditorPacketHandler {

    private final AssetEditorUI editorUI;

    public void handlePopupNotification(AssetEditorPopupNotification packet) {
        // The packet is delivered fully formed by the network layer.
        // The handler's only job is to translate it into a UI action.
        String displayText = formatMessage(packet.message);
        editorUI.showPopup(packet.type, displayText);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Serialization:** Do not call `serialize` or `deserialize` directly. These methods are for internal use by the protocol engine. To send a packet, pass the object instance to the appropriate network connection manager.
- **State Reuse:** Do not hold a reference to a packet instance to modify and resend it. These objects are cheap to create and should be treated as immutable once passed to the network layer.
- **Cross-Thread Access:** Do not create a packet on one thread and modify it from another before sending. All population of the packet's fields must be complete before it is handed off for serialization.

## Data Pipeline
The data flow for this packet is unidirectional, from the server's logic to the client's user interface.

> Flow:
> Server-Side Asset Validation Logic -> `new AssetEditorPopupNotification()` -> Network Engine Serialization -> TCP Stream -> Client Network Engine Deserialization -> **AssetEditorPopupNotification** instance -> Packet Dispatcher -> UI Handler -> Render Popup Notification

