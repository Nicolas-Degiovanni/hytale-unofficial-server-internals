---
description: Architectural reference for RequestFlyCameraMode
---

# RequestFlyCameraMode

**Package:** com.hypixel.hytale.protocol.packets.camera
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class RequestFlyCameraMode implements Packet {
```

## Architecture & Concepts
The RequestFlyCameraMode class is a network packet definition within the Hytale protocol layer. It serves as a simple, client-initiated command to toggle a specific server-side player state: the "fly camera" mode. This is a common feature used for debugging, administration, or spectator gameplay.

As a concrete implementation of the Packet interface, this class is a fundamental data structure for client-server communication. Its design prioritizes efficiency and minimal overhead. The packet has a fixed size of a single byte, making it extremely lightweight and suitable for real-time command transmission.

This class is not a service or a manager; it is pure data. It represents a single, atomic event—a request—that is created, serialized, sent over the network, deserialized, and then immediately discarded. The server-side packet handler is responsible for interpreting this request and applying the corresponding change to the player's game state.

## Lifecycle & Ownership
- **Creation:** Instantiated on the client by the input handling system. This typically occurs when a user presses a specific keybind assigned to toggle flying. A new instance is created for each distinct request.
- **Scope:** The object's lifetime is exceptionally short. On the client, it exists only long enough to be constructed and passed to the network layer for serialization. On the server, it is created during deserialization, processed by a handler, and then becomes immediately eligible for garbage collection.
- **Destruction:** The object is managed by the Java garbage collector. Due to its brief lifecycle and lack of external references post-processing, it is reclaimed very quickly.

## Internal State & Concurrency
- **State:** The internal state is mutable and consists of a single boolean field, *entering*. This field dictates whether the request is to enable or disable the fly camera mode. While technically mutable, the object is intended to be treated as immutable after its initial construction.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container and provides no internal synchronization. It is designed to be created, populated, and processed within a single thread, such as a client-side game loop thread or a server-side Netty event loop thread. Concurrent modification from multiple threads will lead to undefined behavior.

## API Surface
The public API is focused on serialization, deserialization, and metadata for the network protocol.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | RequestFlyCameraMode | O(1) | **Static Factory.** Constructs a new packet by reading a single byte from the provided buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Encodes the *entering* state as a single byte (1 for true, 0 for false) and writes it to the buffer. |
| computeSize() | int | O(1) | Returns the constant size of the packet payload, which is always 1 byte. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (282). |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static Validator.** Checks if the buffer contains at least 1 readable byte at the given offset. |

## Integration Patterns

### Standard Usage
This packet is created and sent through a high-level network service or connection manager. The developer should never interact directly with the Netty ByteBuf for sending.

```java
// Client-side code, typically in an input handler
boolean shouldStartFlying = true;
RequestFlyCameraMode packet = new RequestFlyCameraMode(shouldStartFlying);

// The network service handles serialization and transmission
clientConnection.sendPacket(packet);
```

### Anti-Patterns (Do NOT do this)
- **State Modification After Queuing:** Modifying the packet's state after it has been passed to the network service is a severe anti-pattern. The network layer may read the object's state from a different thread, leading to race conditions where the sent data does not match the final state.
- **Packet Reuse:** These objects are extremely lightweight and should **never** be cached or reused. Always create a new instance for each command to ensure thread safety and state integrity.
- **Manual Serialization/Deserialization:** Application-level code should not call serialize or deserialize. This is the exclusive responsibility of the protocol codec within the Netty pipeline.

## Data Pipeline
The flow of this data object is unidirectional from the client to the server.

> Flow:
> Client Input Event -> **new RequestFlyCameraMode()** -> Network Service -> Protocol Encoder -> Server Protocol Decoder -> Packet Handler -> Player State Mutation

