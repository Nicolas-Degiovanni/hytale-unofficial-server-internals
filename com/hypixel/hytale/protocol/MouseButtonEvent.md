---
description: Architectural reference for MouseButtonEvent
---

# MouseButtonEvent

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class MouseButtonEvent {
```

## Architecture & Concepts
The MouseButtonEvent class is a Data Transfer Object (DTO) that represents a single, discrete mouse button action within the Hytale engine. It is a fundamental component of the client-side input system and the network protocol layer.

This class is not a service or manager; it is a pure data container. Its design is heavily optimized for high-performance serialization and deserialization directly to and from a Netty ByteBuf. The presence of static fields like FIXED_BLOCK_SIZE and MAX_SIZE, along with the static deserialize and validateStructure methods, indicates its role as a well-defined data contract for network communication or file I/O. It acts as a structured message, translating a low-level hardware event into a format that the game logic and network stack can understand and process efficiently.

## Lifecycle & Ownership
- **Creation:** An instance is created by the client's input handling system whenever the operating system reports a mouse button press or release. It can also be instantiated by the network protocol layer's deserializer when reading an incoming data stream.
- **Scope:** The object's lifetime is exceptionally short. It is designed to be ephemeral, existing only for the duration of processing within a single frame or network packet handler.
- **Destruction:** The object holds no external resources and does not require explicit cleanup. It becomes eligible for garbage collection as soon as it goes out of scope, typically after being processed by an event bus or serialized to a network buffer.

## Internal State & Concurrency
- **State:** The state of MouseButtonEvent is entirely mutable. Its public fields can be modified directly after instantiation. This pattern is common for DTOs that are populated by a system component before being dispatched for consumption. The object's state consists of the button type, its state (pressed/released), and the number of clicks.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. It is designed to be created, populated, and consumed within a single, well-defined thread context, such as the main game loop thread or a Netty I/O worker thread.

**WARNING:** Sharing a MouseButtonEvent instance across threads without external locking or synchronization will lead to race conditions and unpredictable behavior. Do not store instances in shared collections accessible by multiple threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | MouseButtonEvent | O(1) | **Static.** Constructs a new event by reading a fixed block of bytes from a buffer. |
| serialize(ByteBuf) | void | O(1) | Writes the event's state into the provided buffer according to the protocol contract. |
| computeSize() | int | O(1) | Returns the fixed size (3 bytes) of the serialized event. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static.** Checks if a buffer contains enough readable bytes to deserialize the event. |
| clone() | MouseButtonEvent | O(1) | Creates a shallow copy of the event object. |

## Integration Patterns

### Standard Usage
A MouseButtonEvent is typically created by a low-level input system and dispatched via an event bus to interested listeners, such as the UI or player controller systems.

```java
// Example: In an input processing loop
void onMouseButtonAction(MouseButtonType type, MouseButtonState state, byte clickCount) {
    MouseButtonEvent event = new MouseButtonEvent(type, state, clickCount);
    
    // Dispatch to the game's event bus for systems to consume
    gameContext.getEventBus().post(event);
}
```

### Anti-Patterns (Do NOT do this)
- **Object Reuse:** Do not modify and re-dispatch the same MouseButtonEvent instance for a new input action. This can cause severe bugs in systems that process events asynchronously. Always create a new instance for each discrete event.
- **Cross-Thread Access:** Do not create an event on one thread and process it on another without proper synchronization. For example, do not pull an event from a shared queue and modify it while another thread could be reading it.
- **Manual Serialization:** Do not attempt to manually write the event's fields to a ByteBuf. Always use the provided serialize method to guarantee correctness and forward compatibility with the network protocol.

## Data Pipeline
The MouseButtonEvent serves as a data packet in the client's input processing pipeline.

> Flow:
> OS Input Event -> Hytale Input Poller -> **MouseButtonEvent (Instantiation)** -> Game Event Bus -> UI System / Player Controller -> Network Packet Serializer -> Outbound ByteBuf

