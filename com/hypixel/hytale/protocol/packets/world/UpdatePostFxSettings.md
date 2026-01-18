---
description: Architectural reference for UpdatePostFxSettings
---

# UpdatePostFxSettings

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Transient

## Definition
```java
// Signature
public class UpdatePostFxSettings implements Packet {
```

## Architecture & Concepts
The UpdatePostFxSettings class is a network packet, a fundamental Data Transfer Object (DTO) within the Hytale protocol layer. Its sole purpose is to transmit post-processing effect parameters from the server to the client. This allows the server to dynamically control a client's visual experience, for instance, by intensifying bloom in a magical area or adjusting sunshafts based on time of day.

As an implementation of the Packet interface, it adheres to a strict contract for serialization and deserialization required by the network engine. The class itself contains no logic; it is a pure data container. The static fields, such as PACKET_ID and FIXED_BLOCK_SIZE, provide metadata that allows the protocol dispatcher to perform highly efficient, reflection-free packet handling. This design is critical for performance in a real-time game environment.

### Lifecycle & Ownership
- **Creation:**
    - **Sending (Server-Side):** Instantiated directly via its constructor (`new UpdatePostFxSettings(...)`) by a server-side system that needs to dictate client-side rendering effects.
    - **Receiving (Client-Side):** Instantiated by the network protocol layer. The static `deserialize` method is invoked by a central packet dispatcher which identifies the incoming data stream by its PACKET_ID.
- **Scope:** Extremely short-lived. An instance exists only for the brief moment of its transmission or processing. On the sender, it is typically garbage collected after serialization. On the receiver, it is garbage collected after its data has been consumed by the rendering engine.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or manual cleanup procedures associated with this object.

## Internal State & Concurrency
- **State:** The object's state is composed of five public float fields. It is fully **Mutable**. There is no internal caching or derived state.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data structure with no synchronization primitives. It is designed to be created, populated, and serialized on a single thread, or deserialized and consumed on a single thread. Concurrent access will lead to data corruption and unpredictable behavior.

**WARNING:** Do not share instances of this packet across threads. If data must be passed between threads, create a deep copy or pass the primitive values.

## API Surface
The public contract is designed for interaction with the network protocol engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the constant network identifier for this packet type (361). |
| serialize(ByteBuf buf) | void | O(1) | Writes the 20 bytes of float data into the provided Netty buffer in Little Endian format. |
| deserialize(ByteBuf buf, int offset) | static UpdatePostFxSettings | O(1) | Reads 20 bytes from the buffer at a given offset and constructs a new packet instance. |
| computeSize() | int | O(1) | Returns the fixed size of the packet payload, which is always 20 bytes. |
| validateStructure(ByteBuf buffer, int offset) | static ValidationResult | O(1) | Performs a pre-deserialization check to ensure the buffer contains enough readable bytes. |

## Integration Patterns

### Standard Usage
This packet is not intended to be used directly by most game logic developers. It is handled by the underlying network and rendering systems.

**Sending (Server-Side Example)**
```java
// A server-side system (e.g., WorldManager) creates and sends the packet
UpdatePostFxSettings newFx = new UpdatePostFxSettings(1.0f, 0.8f, 1.2f, 1.5f, 0.9f);
playerConnection.sendPacket(newFx);
```

**Receiving (Client-Side Handler Stub)**
```java
// The client's PacketHandler receives the deserialized object
public void handle(UpdatePostFxSettings packet) {
    PostFxManager fxManager = client.getRenderingEngine().getPostFxManager();
    fxManager.applySettings(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not hold onto a packet instance to send it multiple times. They are cheap to create and should be treated as immutable messages once sent.
- **Manual Serialization:** Avoid calling `serialize` directly. The network layer wraps packets with additional metadata. Always use a higher-level API like `playerConnection.sendPacket()`.
- **Partial Deserialization:** Do not attempt to read fields from the `ByteBuf` manually. Always use the static `deserialize` method to ensure correct byte order and object construction.

## Data Pipeline
The flow of this data is unidirectional, from the server's game state to the client's GPU.

> **Server Flow:**
> Game State Change -> `new UpdatePostFxSettings(...)` -> Network Engine -> **serialize()** -> TCP/UDP Stream

> **Client Flow:**
> TCP/UDP Stream -> Netty ByteBuf -> Packet Dispatcher (ID 361) -> **deserialize()** -> UpdatePostFxSettings Instance -> Rendering System -> GPU Shader Uniform Update

