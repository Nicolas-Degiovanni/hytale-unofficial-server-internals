---
description: Architectural reference for MouseMotionEvent
---

# MouseMotionEvent

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class MouseMotionEvent {
```

## Architecture & Concepts
The MouseMotionEvent class is a specialized Data Transfer Object (DTO) designed for high-performance network communication. It represents a single, discrete user input event related to mouse movement and button state. This class is a fundamental component of the client-to-server input protocol.

Its primary architectural role is to serve as a structured, serializable container that translates raw user input on the client into a binary format for network transmission, and vice-versa on the server. The design heavily prioritizes network efficiency and deserialization speed. This is achieved through several key patterns:

*   **Bitmasking for Nulls:** A single byte, `nullBits`, is used as a bitfield to indicate the presence or absence of nullable fields. This avoids the overhead of sending null markers for each optional field.
*   **Fixed and Variable Blocks:** The serialized structure is divided into a fixed-size block for predictable data (like `relativeMotion`) and a variable-size block for dynamic data (like the `mouseButtonType` array). This allows for extremely fast parsing, as the position of fixed-size data is always known.
*   **VarInt Encoding:** The length of the `mouseButtonType` array is encoded using a variable-length integer (VarInt). This minimizes the bytes used for small array counts, which is the common case.

This class is not a service or manager; it is pure data. It is created, populated, serialized, and then discarded.

## Lifecycle & Ownership
- **Creation:** MouseMotionEvent instances are created under two primary circumstances:
    1.  **Client-Side (Input Capture):** Instantiated by the client's input handling system each time a mouse movement event is detected. The constructor is called with the current relative motion and button states.
    2.  **Server-Side (Packet Deserialization):** Instantiated within the network protocol layer by the static `deserialize` factory method when an incoming network packet corresponding to this event is read from a Netty ByteBuf.

- **Scope:** The object's lifetime is exceptionally short and tied to a single network or game tick. It is a transient object, designed to be processed and then immediately eligible for garbage collection.

- **Destruction:** Managed entirely by the Java Garbage Collector. There are no unmanaged resources, and no explicit `close` or `destroy` method is required.

## Internal State & Concurrency
- **State:** The class is a mutable data container. Its public fields, `mouseButtonType` and `relativeMotion`, can be directly modified after instantiation. This design choice prioritizes performance by avoiding the overhead of getters and setters for a short-lived object. It holds no internal caches or derived state.

- **Thread Safety:** **This class is not thread-safe.** It contains no locks or other concurrency primitives. It is designed to be created, written to, and read from within a single thread, such as a Netty I/O worker or the main game logic thread.

    **Warning:** Sharing a MouseMotionEvent instance across threads without external synchronization will result in undefined behavior and is a critical programming error. Always create a new instance or perform a deep copy (using `clone`) if data must be passed between threads.

## API Surface
The public API is focused on serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static MouseMotionEvent | O(N) | Constructs a new MouseMotionEvent by reading from a binary buffer. N is the number of mouse buttons. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into a binary buffer for network transmission. |
| computeSize() | int | O(1) | Calculates the number of bytes this object will occupy when serialized. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a lightweight check on a buffer to ensure it contains a structurally valid event without full deserialization. |
| clone() | MouseMotionEvent | O(N) | Creates a deep copy of the object. |

## Integration Patterns

### Standard Usage
The class is used differently on the sending (client) and receiving (server) ends of the connection.

**Client-Side: Creating and Sending an Event**
```java
// In the client's input processing loop
Vector2i motion = new Vector2i(deltaX, deltaY);
MouseButtonType[] buttons = { MouseButtonType.LEFT };

MouseMotionEvent event = new MouseMotionEvent(buttons, motion);

// The network layer will then take this event and serialize it
// networkChannel.write(event); which internally calls event.serialize(buffer);
```

**Server-Side: Receiving and Processing an Event**
```java
// In a Netty channel handler or protocol dispatcher
ByteBuf incomingPacket = ...;
MouseMotionEvent event = MouseMotionEvent.deserialize(incomingPacket, 0);

// Pass the event to the game logic for processing
player.getController().onMouseMove(event);
```

### Anti-Patterns (Do NOT do this)
- **Object Reuse:** Do not modify and reuse a MouseMotionEvent instance across multiple game ticks. They are cheap to allocate, and reusing them can lead to subtle bugs where old state is accidentally processed.
- **Cross-Thread Modification:** Do not deserialize an event on a network thread and then modify it on the main game thread. Treat the object as immutable once it has been passed to a consumer.
- **Ignoring Validation:** Do not call `deserialize` on untrusted data without first calling `validateStructure`. A malformed packet could throw an exception or cause a buffer overflow, leading to a server crash.

## Data Pipeline
The flow of this data object is unidirectional from the client's hardware to the server's game logic.

> Flow:
> Client OS Input -> Hytale Input System -> **new MouseMotionEvent()** -> Network Encoder (`serialize`) -> TCP/UDP Packet -> Server Network Decoder (`deserialize`) -> Player Input Controller

