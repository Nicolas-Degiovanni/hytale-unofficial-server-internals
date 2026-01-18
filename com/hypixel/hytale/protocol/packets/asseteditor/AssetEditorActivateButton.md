---
description: Architectural reference for AssetEditorActivateButton
---

# AssetEditorActivateButton

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class AssetEditorActivateButton implements Packet {
```

## Architecture & Concepts
The AssetEditorActivateButton is a Data Transfer Object (DTO) that represents a specific, granular command within the Hytale network protocol. It is not a service or a manager; it is an inert data structure whose sole purpose is to convey the intent to activate a UI element, identified by its string ID, within the in-game asset editor.

This class embodies the protocol's design philosophy of creating highly specific, lightweight packet types for each distinct user action. It acts as a contract between the client and server, ensuring that both ends can deterministically serialize and deserialize the command. The implementation leverages low-level, performance-oriented components like Netty's ByteBuf and custom VarInt encoding to minimize network overhead.

Architecturally, it sits at the boundary between the raw network transport layer and the application's business logic. The network layer is responsible for invoking its static factory method, deserialize, upon identifying its unique PACKET_ID (335) in the byte stream. The resulting object is then passed up to a dedicated packet handler for processing.

## Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1. **Inbound (Decoding):** The network protocol decoder instantiates the object by calling the static `deserialize` method when an incoming data frame with ID 335 is detected.
    2. **Outbound (Encoding):** Application logic, typically a UI event handler within the asset editor, creates a new instance via its constructor (`new AssetEditorActivateButton("button_id")`) to prepare a command for transmission.

- **Scope:** The object's lifetime is exceptionally brief and tied to a single network event. It is created, processed, and then immediately becomes eligible for garbage collection. It should never be cached or held for long-term reference.

- **Destruction:** Managed entirely by the Java Garbage Collector. Once the packet handler completes its execution, all references to the packet object are typically dropped, and it is reclaimed during the next GC cycle.

## Internal State & Concurrency
- **State:** The class holds a single, mutable field: the nullable String `buttonId`. All other fields are static final constants that define the packet's structure and identity within the protocol. The object's state is entirely self-contained.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be created and processed within the confines of a single network event loop thread (e.g., a Netty EventLoop). Concurrent modification of the `buttonId` field would lead to unpredictable behavior and data corruption during serialization. The protocol's single-threaded processing model per connection inherently prevents these concurrency issues in standard usage.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet type (335). |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided ByteBuf. Complexity is proportional to the length of `buttonId`. |
| deserialize(ByteBuf, int) | AssetEditorActivateButton | O(N) | Static factory. Decodes a new instance from the ByteBuf at the given offset. Throws ProtocolException on malformed data. |
| computeSize() | int | O(N) | Calculates the number of bytes required to serialize the object. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Static utility. Performs a read-only check of the buffer to validate if the data represents a valid packet without full deserialization. |

## Integration Patterns

### Standard Usage
The primary integration pattern involves a dedicated packet handler that receives the fully-formed object from the network layer. The handler extracts the necessary data and delegates the action to the appropriate system.

```java
// Example of a packet handler receiving the object
public class AssetEditorPacketHandler {
    private final AssetEditorUIService uiService;

    public void handle(AssetEditorActivateButton packet) {
        String button = packet.buttonId;
        if (button != null) {
            // Delegate the action to the relevant UI service
            uiService.onActivateButton(button);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not cache and re-use packet instances. They are cheap to create and are not designed for multiple serialization cycles, especially if their internal state is modified. Always create a new instance for each outbound message.
- **Cross-Thread Access:** Never pass a packet instance to another thread for processing. If business logic must be executed off the network thread, extract the primitive data (`buttonId`) and pass that instead.
- **Manual Deserialization:** Do not attempt to manually parse the ByteBuf. Always rely on the static `deserialize` method, as it correctly handles the protocol's specific encoding rules, such as null-bit fields and VarInts.

## Data Pipeline
The AssetEditorActivateButton serves as a data record that flows through the network stack. Its journey is linear and unidirectional for any single event.

> **Inbound Flow (Client -> Server):**
> Raw TCP Bytes -> Netty Channel -> Protocol Frame Decoder -> Packet Dispatcher (reads ID 335) -> `AssetEditorActivateButton.deserialize` -> **AssetEditorActivateButton Instance** -> Application Packet Handler -> Asset Editor Service

> **Outbound Flow (Server -> Client):**
> Application Logic -> `new AssetEditorActivateButton()` -> Network Service -> `AssetEditorActivateButton.serialize` -> Netty Channel -> Raw TCP Bytes

