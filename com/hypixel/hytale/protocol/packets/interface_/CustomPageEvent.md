---
description: Architectural reference for CustomPageEvent
---

# CustomPageEvent

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class CustomPageEvent implements Packet {
```

## Architecture & Concepts
The CustomPageEvent class is a Data Transfer Object (DTO) that represents a specific message within Hytale's network protocol, identified by the static packet ID 219. It is not a service or a manager; it is a pure data container designed for serialization and network transport.

This packet facilitates communication between the client and server regarding custom user interfaces. Its primary purpose is to carry a payload, typically a serialized string like JSON, which describes a UI event or state change. The class encapsulates the logic for its own binary encoding and decoding, making it a self-contained component of the network protocol layer.

The binary layout is explicitly defined through internal constants such as FIXED_BLOCK_SIZE and NULLABLE_BIT_FIELD_SIZE. It employs a common network optimization pattern using a bitmask, referred to as *nullBits*, to efficiently indicate the presence or absence of optional fields like the *data* string, minimizing payload size for events that do not require extra data.

## Lifecycle & Ownership
- **Creation:** An instance of CustomPageEvent is created under two primary circumstances:
    1.  **Inbound:** The network layer's packet dispatcher instantiates it by calling the static *deserialize* method when an incoming network buffer with packet ID 219 is processed.
    2.  **Outbound:** Game logic, such as a UI script or a server-side plugin, creates a new instance via its constructor to send an event to the remote peer.

- **Scope:** The object's lifetime is extremely short and scoped to a single transaction. It is created, processed by a handler or written to a network channel, and then immediately becomes eligible for garbage collection.

- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or `close` methods. References should not be held beyond the immediate processing scope.

## Internal State & Concurrency
- **State:** The class is mutable. Its public fields, *type* and *data*, can be modified after construction. It is designed as a simple, transient data holder.

- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. Instances are intended for use within a single thread, typically a Netty I/O thread or the main game logic thread.

    **Warning:** Sharing a CustomPageEvent instance across multiple threads without external synchronization will result in race conditions and undefined behavior. Do not write to an instance from one thread while another thread is reading it or serializing it.

## API Surface
The primary contract is defined by its static factory and instance serialization methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static CustomPageEvent | O(N) | Constructs a new CustomPageEvent by reading from a ByteBuf. Throws ProtocolException on malformed data. N is the length of the data string. |
| serialize(buf) | void | O(N) | Encodes the object's state into the provided ByteBuf for network transmission. N is the length of the data string. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a lightweight, non-deserializing check on a buffer to verify if it contains a structurally valid packet. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. N is the length of the data string. |

## Integration Patterns

### Standard Usage
Developers should never need to call serialization or deserialization methods directly. The network engine handles this. The standard pattern is to create an instance and pass it to the network layer for dispatch.

```java
// Example: Sending an event from game logic
CustomPageEvent event = new CustomPageEvent(CustomPageEventType.Update, "{ \"key\": \"value\" }");

// The network system handles the actual serialization and sending
connection.sendPacket(event);
```

On the receiving end, a handler would be registered to listen for this specific packet type.

```java
// Example: A hypothetical packet handler
public void onCustomPageEvent(CustomPageEvent event) {
    if (event.type == CustomPageEventType.Update) {
        // Process the JSON data from event.data
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not modify and re-send the same CustomPageEvent instance. They are lightweight and should be treated as immutable messages once created. Create a new instance for each message.
- **Manual Buffer Management:** Avoid calling *serialize* or *deserialize* directly. The network system provides higher-level abstractions like `sendPacket` and packet listeners that manage the underlying byte buffers and protocol framing.
- **Cross-Thread Sharing:** Never pass an instance from the network thread to a worker thread without creating a defensive copy, as the original network buffer may be recycled.

## Data Pipeline
The CustomPageEvent acts as a data payload that flows through the network stack.

**Outbound (Sending an Event):**
> Flow:
> Game Logic -> `new CustomPageEvent()` -> Network System -> **CustomPageEvent::serialize** -> Netty Channel -> Encoded Byte Stream

**Inbound (Receiving an Event):**
> Flow:
> Raw Byte Stream -> Netty Channel -> Packet Framer -> **CustomPageEvent::deserialize** -> Packet Dispatcher -> Event Handler Logic

