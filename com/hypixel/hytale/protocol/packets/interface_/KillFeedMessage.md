---
description: Architectural reference for KillFeedMessage
---

# KillFeedMessage

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient

## Definition
```java
// Signature
public class KillFeedMessage implements Packet {
```

## Architecture & Concepts
The KillFeedMessage class is a Data Transfer Object (DTO) that represents a single entry in the in-game kill feed. It is a fundamental component of the client-server communication protocol, designed exclusively for transmitting structured data, not for encapsulating application logic.

As an implementation of the Packet interface, its primary role is to define the binary layout for a specific network message (ID 213). The design heavily favors performance and efficiency by employing static methods for core network operations like deserialization and validation. This allows the network pipeline, typically managed by Netty, to process raw byte buffers and construct KillFeedMessage objects without prior instantiation, minimizing object allocation overhead.

The structure of this packet is optimized for variable-sized data. It uses a fixed-size header containing a bitmask (nullBits) and offsets to a variable-sized data block. This allows for optional fields without wasting network bandwidth, a critical consideration for real-time game traffic.

### Lifecycle & Ownership
- **Creation:**
    - **Sending Peer (e.g., Server):** Instantiated directly via its constructor (`new KillFeedMessage(...)`) by game logic when a kill event occurs. The object is then passed to the network layer for serialization.
    - **Receiving Peer (e.g., Client):** Never instantiated directly. The object is created by the network framework by invoking the static `deserialize` method on a raw ByteBuf.
- **Scope:** Extremely short-lived. A KillFeedMessage instance exists only for the brief period between its creation and its consumption. On the sender, it is serialized and immediately becomes eligible for garbage collection. On the receiver, it is deserialized and passed to a handler, after which it is also eligible for collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. This class holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The KillFeedMessage is a mutable data container. Its public fields can be modified after construction. The state consists of two FormattedMessage objects and a String, representing the killer, the decedent, and the weapon or method icon.
- **Thread Safety:** This class is **not thread-safe** and must not be shared between threads without external synchronization. It is designed to be created, processed, and discarded within the context of a single thread, such as a Netty event loop thread or a game tick thread.

**WARNING:** Concurrent modification of a KillFeedMessage instance will lead to unpredictable behavior and data corruption. It should be treated as an immutable object after being passed to another system or thread.

## API Surface
The public API is divided between instance methods for serialization and static methods for deserialization and validation from a raw byte stream.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf buf) | void | O(N) | Encodes the object's state into the provided Netty ByteBuf according to the defined binary protocol. |
| deserialize(ByteBuf buf, int offset) | static KillFeedMessage | O(N) | Decodes a KillFeedMessage from the given ByteBuf at a specific offset. This is the primary entry point for packet creation on the receiving end. |
| validateStructure(ByteBuf buf, int offset) | static ValidationResult | O(N) | Performs a series of checks on a raw buffer to ensure it contains a valid, non-corrupt KillFeedMessage without performing a full deserialization. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the current state of the object. Used for pre-allocating buffers. |
| computeBytesConsumed(ByteBuf buf, int offset) | static int | O(N) | Calculates the total size of a serialized packet directly from a buffer, which is essential for advancing the buffer's read position. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol layer. A packet dispatcher identifies the packet ID from the stream and delegates the buffer to the static `deserialize` method.

```java
// Executed within a network pipeline handler
// byteBuf contains incoming network data

// Assume packetId has been read and is 213
if (packetId == KillFeedMessage.PACKET_ID) {
    ValidationResult result = KillFeedMessage.validateStructure(byteBuf, byteBuf.readerIndex());
    if (!result.isValid()) {
        // Handle corrupt packet, possibly disconnect client
        throw new ProtocolException(result.error());
    }

    KillFeedMessage message = KillFeedMessage.deserialize(byteBuf, byteBuf.readerIndex());
    int bytesConsumed = KillFeedMessage.computeBytesConsumed(byteBuf, byteBuf.readerIndex());
    byteBuf.skipBytes(bytesConsumed);

    // Pass the fully formed message to the UI event bus or game state manager
    gameContext.getEventBus().post(message);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation on Receiver:** Do not call `new KillFeedMessage()` on the client or receiving peer. The object must be constructed from the network stream using `KillFeedMessage.deserialize` to ensure data integrity.
- **State Reuse:** Do not hold a reference to a KillFeedMessage object across multiple game ticks or events. It represents a point-in-time event and should be processed and discarded immediately.
- **Asynchronous Modification:** Do not modify a KillFeedMessage object after it has been submitted to the network layer for serialization. This will cause a race condition where the object might be serialized in a partially updated, invalid state.

## Data Pipeline
The KillFeedMessage acts as a data payload that flows through the network stack. Its primary function is to carry UI event data from the server's game logic to the client's rendering engine.

> Flow:
> Server Game Event -> **new KillFeedMessage()** -> serialize() -> Netty Channel -> Client Network Handler -> **deserialize()** -> UI Event Bus -> Kill Feed UI Render

