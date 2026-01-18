---
description: Architectural reference for BuilderToolRotateClipboard
---

# BuilderToolRotateClipboard

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient Data Object

## Definition
```java
// Signature
public class BuilderToolRotateClipboard implements Packet {
```

## Architecture & Concepts
The BuilderToolRotateClipboard class is a Data Transfer Object (DTO) that represents a specific, atomic command within the Hytale network protocol. It is not a service or manager, but rather a structured message encapsulating a user's intent to rotate the contents of their in-game builder clipboard.

This packet is designed for high-performance network communication. Its structure is fixed-size, containing only an integer for the angle and a byte for the rotation axis. This design eliminates the need for complex parsing or dynamic memory allocation during the critical serialization and deserialization phases, which typically occur on performance-sensitive network threads.

As a component of the protocol, it serves as a rigid data contract between the client and server. Any modification to its layout, size, or field order constitutes a breaking change to the protocol version. It is a fundamental building block for the server's remote procedure call (RPC) system for in-game creative tools.

## Lifecycle & Ownership
- **Creation:** An instance is created on the client when a user initiates a rotation action via the UI. On the server, an instance is created by the protocol's deserialization layer when a network buffer with packet ID 406 is processed.
- **Scope:** The object's lifetime is extremely brief and transactional. On the client, it exists only for the duration of the serialization process. On the server, it exists from the moment of deserialization until it is consumed by the designated game logic handler. It is then immediately eligible for garbage collection.
- **Destruction:** This object is managed by the Java garbage collector. It is dereferenced and becomes eligible for collection as soon as the network or game logic that created it completes its unit of work. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The internal state is fully mutable via public fields. This design choice prioritizes raw performance and low allocation overhead over encapsulation, which is a common and acceptable trade-off for internal, short-lived DTOs.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. It is designed to be created, populated, and processed within the context of a single thread, such as a Netty I/O worker or a main game-loop thread.

**WARNING:** Sharing an instance of BuilderToolRotateClipboard across multiple threads without external synchronization is a severe programming error and will result in data corruption and unpredictable system behavior.

## API Surface
The primary API is static and focused on protocol-level operations, managed by the network engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | BuilderToolRotateClipboard | O(1) | **[Static Factory]** Constructs a new instance from a raw network buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the instance's state into the provided network buffer. |
| computeSize() | int | O(1) | Returns the constant size of the packet in bytes. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **[Static]** Verifies if a buffer has sufficient bytes to contain this packet. |

## Integration Patterns

### Standard Usage
This packet is not intended for direct use by most game feature developers. It is created and consumed by the low-level UI and network systems.

A typical client-side flow involves creating the packet in response to user input and submitting it to the network layer for dispatch.

```java
// Client-side example within a UI event handler
Axis rotationAxis = Axis.Y;
int rotationAngle = 90;

BuilderToolRotateClipboard packet = new BuilderToolRotateClipboard(rotationAngle, rotationAxis);
clientNetworkManager.sendPacket(packet);
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not hold references to packet instances for reuse. They are designed to be cheap, single-use objects. Reusing them can lead to sending stale or incorrect data.
- **Direct Serialization:** Avoid calling serialize or deserialize directly in application-level code. These methods are the exclusive responsibility of the core protocol engine, which handles buffering, framing, and dispatch.
- **Modification After Send:** Do not modify a packet's fields after it has been passed to the network manager. The serialization may occur on a different thread, leading to a race condition.

## Data Pipeline
This packet follows a unidirectional client-to-server data flow. It represents a command from the user to the game server.

> Flow:
> Client UI Input -> Input Handler -> **new BuilderToolRotateClipboard(...)** -> Network Manager -> `serialize()` -> Netty Channel -> Server Network Listener -> Packet Dispatcher -> `deserialize()` -> Builder Tool Service -> World State Update

