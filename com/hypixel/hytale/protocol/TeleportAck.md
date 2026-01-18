---
description: Architectural reference for TeleportAck
---

# TeleportAck

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class TeleportAck {
```

## Architecture & Concepts
The TeleportAck class is a network protocol message, representing a client's or server's acknowledgment of a teleportation event. It serves as a simple, fixed-size data container designed for high-performance network communication.

This class is a fundamental component of the low-level network layer, acting as a concrete representation of a packet that flows through the Netty pipeline. Its design prioritizes minimal overhead:

*   **Fixed Structure:** The packet has a constant size of 1 byte, defined by the static field FIXED_BLOCK_SIZE. This allows for highly predictable buffer allocation and parsing.
*   **Direct Serialization:** It contains explicit serialization and deserialization logic that operates directly on Netty ByteBufs, avoiding intermediate representations or reflection-based frameworks.
*   **Value Semantics:** While technically mutable, it is intended to be used as an immutable value object. Once created, its state represents a single, atomic network message.

The presence of static constants like MAX_SIZE and VARIABLE_FIELD_COUNT indicates that this class is part of a larger, possibly code-generated, protocol definition framework that standardizes packet structure and validation.

## Lifecycle & Ownership
- **Creation:**
    - **Sending Peer:** An instance is created via its constructor, `new TeleportAck(id)`, when the game logic needs to send a teleport acknowledgment.
    - **Receiving Peer:** An instance is materialized by the network protocol dispatcher, which calls the static factory method `TeleportAck.deserialize` on an incoming network ByteBuf.
- **Scope:** Extremely short-lived and transient. An instance exists only for the duration of its processing within a single network event handler. It is a message, not a persistent entity.
- **Destruction:** The object becomes eligible for garbage collection as soon as the event handler that created or received it completes. There are no manual cleanup requirements.

## Internal State & Concurrency
- **State:** Mutable. The class contains a single public field, `teleportId`, which can be modified directly. However, standard usage patterns treat the object as immutable after construction.
- **Thread Safety:** **This class is not thread-safe.** The public, mutable field makes it unsafe for concurrent access. It is designed exclusively for use within a single-threaded context, such as a Netty I/O event loop.

    **WARNING:** Never share an instance of TeleportAck across threads. If the teleportId value is needed by another thread, copy the primitive value into a thread-safe data structure.

## API Surface
The public API is focused on construction, serialization, and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TeleportAck(byte) | constructor | O(1) | Constructs a new packet with the specified ID. |
| deserialize(ByteBuf, int) | static TeleportAck | O(1) | Constructs a TeleportAck instance by reading 1 byte from the buffer. |
| serialize(ByteBuf) | void | O(1) | Writes the teleportId into the provided buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if the buffer has enough readable bytes to contain this packet. |
| computeSize() | int | O(1) | Returns the fixed size of the packet, which is always 1. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol layer. A dispatcher identifies the packet type from an incoming stream and uses the static `deserialize` method to construct the object.

```java
// Example within a hypothetical Netty channel handler
ByteBuf incomingBuffer = ...;
int offset = ...; // Start of the packet in the buffer

// Validate that the buffer can contain the packet
ValidationResult result = TeleportAck.validateStructure(incomingBuffer, offset);
if (!result.isOk()) {
    // Handle error: close connection or log violation
    return;
}

// Deserialize the packet
TeleportAck ack = TeleportAck.deserialize(incomingBuffer, offset);

// Pass the DTO to the game logic layer for processing
gameEventHandler.onTeleportAcknowledged(ack.teleportId);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not maintain references to TeleportAck instances in caches, game state, or other long-lived objects. They are transient messages. Extract the `teleportId` value and discard the container object.
- **State Mutation:** Do not modify the `teleportId` field after deserialization or before serialization. This violates its role as a value object and can lead to unpredictable behavior.
- **Cross-Thread Sharing:** Do not pass a TeleportAck instance from the network thread to a worker thread. This creates a race condition.

## Data Pipeline
The TeleportAck object is a carrier of data between the raw byte stream and the application logic.

> **Receiving Flow:**
> Network Byte Stream -> Netty Channel Pipeline -> Protocol Frame Decoder -> **TeleportAck.deserialize** -> Game Event Handler

> **Sending Flow:**
> Game Event Handler -> **new TeleportAck(id)** -> **serialize(ByteBuf)** -> Netty Channel Pipeline -> Network Byte Stream

