---
description: Architectural reference for InteractionCamera
---

# InteractionCamera

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class InteractionCamera {
```

## Architecture & Concepts
The InteractionCamera class is a high-performance Data Transfer Object (DTO) designed for the Hytale network protocol. It is not a service or a manager; it is a pure data structure that represents the state of a player's camera at the moment of an interaction. Its primary role is to serve as a data contract, ensuring that camera information is serialized and deserialized consistently and efficiently between the client and server.

The design is heavily optimized for low-latency network communication:
- **Fixed-Size Layout:** The object serializes to a constant block of 29 bytes. This predictability eliminates the need for dynamic size calculations or length prefixes, simplifying buffer management and reducing processing overhead.
- **Direct Buffer Manipulation:** Serialization and deserialization logic operates directly on Netty ByteBuf instances, avoiding intermediate data structures or reflection, which is critical for performance in the network layer.
- **Bitmask for Nulls:** A single byte, the nullable bit field, is used as a bitmask to track the presence of optional fields like position and rotation. This is a highly compact method for handling nullable complex types without wasting space.

This class is a fundamental building block for any network packet that needs to convey precise camera orientation and position, such as packets related to block placement, entity interaction, or aiming mechanics.

### Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1.  By the network protocol layer when a packet containing camera data is received and the static deserialize method is called.
    2.  By game logic on the sending side, which constructs and populates an instance before passing it to the network layer for serialization.
- **Scope:** The object is transient and has an extremely short lifecycle. Its scope is typically confined to the processing of a single network packet or a single game tick's logic. It is not intended to be stored or referenced long-term.
- **Destruction:** As a simple heap-allocated object with no external resources, it is managed by the Java garbage collector. It becomes eligible for collection as soon as all references to it are dropped, which usually occurs immediately after the relevant network packet or game event has been fully processed.

## Internal State & Concurrency
- **State:** InteractionCamera is a mutable data container. Its public fields can be modified directly after instantiation. This design prioritizes performance and ease of use within the protocol layer over strict immutability.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. All access and modification must be confined to a single thread, typically a Netty I/O thread or the main game logic thread.

**WARNING:** Sharing an InteractionCamera instance across threads without external, explicit synchronization will lead to race conditions, data corruption, and unpredictable behavior. Do not store instances in shared collections or pass them between threads.

## API Surface
The public API is focused exclusively on serialization, deserialization, and data validation for the network protocol.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | InteractionCamera | O(1) | Constructs a new InteractionCamera by reading 29 bytes from the given buffer at a specific offset. |
| serialize(buf) | void | O(1) | Writes the object's state into the provided buffer as a fixed-size 29-byte block. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object, which is always 29. |
| validateStructure(buffer, offset) | ValidationResult | O(1) | Checks if the buffer contains enough readable bytes (at least 29) to deserialize the object from the given offset. |

## Integration Patterns

### Standard Usage
The class is intended to be used by higher-level network packet codecs. Game logic populates the object, and the network layer handles the serialization.

```java
// Example: Preparing an interaction packet for sending
Vector3f cameraPos = new Vector3f(10.5f, 64.0f, -20.1f);
Direction cameraRot = new Direction(0.707f, 0.0f, 0.707f, 0.0f);

// 1. Create and populate the DTO
InteractionCamera cameraState = new InteractionCamera(
    System.currentTimeMillis() / 1000.0f,
    cameraPos,
    cameraRot
);

// 2. Serialize into a network buffer (typically done by a Packet Codec)
ByteBuf networkBuffer = Unpooled.buffer(InteractionCamera.MAX_SIZE);
cameraState.serialize(networkBuffer);

// 3. The buffer is now ready to be sent over the network.
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not retain and modify an InteractionCamera instance across multiple game ticks or network packets. This can lead to subtle bugs where old state is accidentally sent. Always create a new instance for each distinct network message.
- **Cross-Thread Access:** Never write to an InteractionCamera instance from one thread while another thread might be reading it for serialization. This is a classic race condition.
- **Modification After Submission:** Do not modify an instance after it has been passed to the network layer for sending. The serialization may happen on a different thread, and later modifications can corrupt the outgoing data.

## Data Pipeline
The InteractionCamera acts as a data payload that flows through the network stack. Its representation changes from an in-memory object to a raw byte sequence and back.

> **Outbound (Client -> Server):**
> Game Logic creates **InteractionCamera** object -> Packet Codec calls serialize() -> **InteractionCamera** becomes a 29-byte sequence in a ByteBuf -> Netty sends the ByteBuf to the network socket.

> **Inbound (Server <- Client):**
> Netty receives bytes into a ByteBuf -> Packet Codec calls deserialize() -> The 29-byte sequence becomes an **InteractionCamera** object -> Game Logic processes the object.

