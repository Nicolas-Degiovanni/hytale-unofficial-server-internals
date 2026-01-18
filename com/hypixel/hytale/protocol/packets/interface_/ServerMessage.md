---
description: Architectural reference for ServerMessage
---

# ServerMessage

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class ServerMessage implements Packet {
```

## Architecture & Concepts
The ServerMessage class is a Data Transfer Object (DTO) that represents a specific network packet, identified by the static ID 210. Its sole responsibility is to model a message sent from the server to the client, such as a chat message or a system notification.

As an implementation of the Packet interface, it serves as a fundamental building block of the Hytale network protocol layer. It is not a service or a manager; it is a pure data container with a strictly defined binary layout.

The design is heavily optimized for network efficiency. It employs a bitmask, `nullBits`, to compactly represent the presence of nullable fields, in this case the `message` field. This avoids transmitting unnecessary bytes for optional data. The class provides static methods for deserialization, validation, and size calculation, allowing the network layer to operate on raw byte buffers without needing to instantiate objects prematurely.

## Lifecycle & Ownership
- **Creation:**
    - **Inbound (Client):** Instantiated exclusively by the network protocol decoder, typically a Netty Channel Handler, when an incoming data stream contains a packet with ID 210. The static `deserialize` factory method is the designated entry point.
    - **Outbound (Server):** Instantiated by server-side game logic, such as a chat or notification system, when a message must be transmitted to a client.

- **Scope:** The lifecycle of a ServerMessage instance is extremely short and transactional.
    - On the client, it exists only for the duration of its processing by a packet handler. It is typically passed to a higher-level system (e.g., the UI) and then immediately becomes eligible for garbage collection.
    - On the server, it exists only until it has been serialized into a ByteBuf by the network encoder for transmission.

- **Destruction:** Ownership is never transferred. The object is managed entirely by the Java Garbage Collector and has no explicit `destroy` or `close` methods.

## Internal State & Concurrency
- **State:** Mutable. The fields of a ServerMessage can be modified after construction. However, by convention and design, it should be treated as an immutable value object once it has been deserialized or prepared for serialization.

- **Thread Safety:** **This class is not thread-safe.** It is a simple data container designed for use within a single thread context, such as a Netty I/O thread or the main game thread. Concurrent access and modification from multiple threads will result in data corruption and race conditions. All serialization and deserialization operations must be synchronized externally if multi-threaded access is unavoidable.

## API Surface
The public contract is focused on serialization and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | ServerMessage | O(N) | **Static Factory.** Reads from a ByteBuf and constructs a new ServerMessage instance. |
| serialize(buf) | void | O(N) | Encodes the object's state into the provided ByteBuf for network transmission. |
| computeSize() | int | O(N) | Calculates the exact number of bytes the object will occupy when serialized. |
| validateStructure(buf, offset) | ValidationResult | O(N) | **Static.** Checks if a buffer segment can be successfully parsed as a ServerMessage without full deserialization. |
| getId() | int | O(1) | Returns the static network identifier (210) for this packet type. |

*N = size of the FormattedMessage content.*

## Integration Patterns

### Standard Usage
A packet handler receives a fully-formed ServerMessage object from the network layer. The handler should inspect the message type and content, then dispatch the information to the appropriate game system, such as a UI controller.

```java
// In a client-side packet handler
public void handleServerMessage(ServerMessage packet) {
    FormattedMessage content = packet.message;
    ChatType type = packet.type;

    if (content != null) {
        // Dispatch to the Chat UI system to render the message
        chatUI.displayMessage(type, content.toPlainText());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not modify a ServerMessage instance after it has been received or sent and then attempt to re-process or re-send it. This breaks the transactional nature of network packets and can cause severe bugs, especially in a multi-threaded environment. Always create a new instance for a new message.
- **Manual Deserialization:** Do not attempt to read the fields from a ByteBuf manually. The binary layout is an implementation detail. Always use the static `deserialize` method to ensure forward compatibility and correctness.

## Data Pipeline
The ServerMessage acts as a data record that flows from the network hardware to the game logic.

> **Flow (Client-Side):**
> Network Byte Stream -> Netty Protocol Decoder -> **ServerMessage.deserialize** -> ServerMessage Instance -> Client Packet Router -> Game System (e.g., ChatController) -> UI Update

