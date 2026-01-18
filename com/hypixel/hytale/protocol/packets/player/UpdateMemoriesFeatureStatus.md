---
description: Architectural reference for UpdateMemoriesFeatureStatus
---

# UpdateMemoriesFeatureStatus

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class UpdateMemoriesFeatureStatus implements Packet {
```

## Architecture & Concepts
UpdateMemoriesFeatureStatus is a network packet data structure, not a service or manager. It serves as a concrete implementation of the Packet interface, designed for low-level network communication. Its sole purpose is to transmit a single boolean flag indicating the unlocked status of the "Memories Feature" between the server and client.

This class is a fundamental component of the protocol layer. The static fields, such as PACKET_ID and FIXED_BLOCK_SIZE, provide essential metadata to the protocol framework. This metadata enables the framework to perform highly efficient, zero-reflection serialization and deserialization, dispatching incoming byte streams to the correct packet parser without expensive lookups. It represents a command or a state synchronization event, not a persistent entity.

## Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1.  **Inbound:** The network protocol dispatcher instantiates the object by invoking the static `deserialize` method when an incoming network buffer with packet ID 118 is detected.
    2.  **Outbound:** Server-side or client-side game logic instantiates the object directly via its constructor to prepare a message for transmission.
- **Scope:** The object's lifetime is exceptionally brief. It is designed to be ephemeral, existing only for the duration of its processing within a single network tick or event handler. It should be considered a message, not a state container.
- **Destruction:** The object is immediately eligible for garbage collection once the consuming system (e.g., a packet handler) completes its execution. There are no persistent references to it within the engine.

## Internal State & Concurrency
- **State:** The internal state consists of a single, mutable boolean field: isFeatureUnlocked. While technically mutable, the object is treated as immutable after its initial creation or deserialization. It is a simple data carrier.
- **Thread Safety:** This class is **not thread-safe**. Direct access to its public field from multiple threads would be hazardous. However, the underlying network framework, likely Netty, typically guarantees that packet processing occurs on a single I/O worker thread, preventing concurrent access by design. Any handoff to other threads must be managed explicitly by the consuming system.

## API Surface
The primary contract is defined by the static serialization methods and the Packet interface, not instance methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | UpdateMemoriesFeatureStatus | O(1) | Static factory. Constructs an object from a raw network buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state into a network buffer for transmission. |
| getId() | int | O(1) | Returns the unique, static identifier (118) for this packet type. |
| computeSize() | int | O(1) | Returns the fixed size in bytes (1) that this packet occupies on the wire. |

## Integration Patterns

### Standard Usage
This packet is handled exclusively by the network layer and game event systems. Application logic should not interact with its serialization methods directly.

**Sending a status update (Server-side logic):**
```java
// Example: A game service unlocks the feature and notifies the client.
UpdateMemoriesFeatureStatus packet = new UpdateMemoriesFeatureStatus(true);
playerConnection.sendPacket(packet);
```

**Receiving a status update (Client-side handler):**
```java
// The network layer deserializes the packet and routes it to a handler.
public void handlePacket(UpdateMemoriesFeatureStatus packet) {
    // Update the UI or game state based on the received status.
    MemoriesFeatureManager.setUnlocked(packet.isFeatureUnlocked);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store instances of this packet in caches or as member variables of services. It represents a transient event, and its data should be immediately transferred to a proper state manager.
- **Modification After Receipt:** Never modify the state of a packet object after it has been deserialized. This corrupts the record of the event that occurred on the network.
- **Manual Invocation:** Do not call `serialize` or `deserialize` directly. These methods are strictly for use by the core protocol engine.

## Data Pipeline
The class acts as a data container that is transformed to and from a byte stream at the boundaries of the network stack.

> **Outbound Flow:**
> Game Logic -> `new UpdateMemoriesFeatureStatus()` -> Network Engine -> `serialize()` -> Raw ByteBuf on TCP Stream

> **Inbound Flow:**
> Raw ByteBuf from TCP Stream -> Protocol Dispatcher -> `deserialize()` -> **UpdateMemoriesFeatureStatus Instance** -> Event Bus -> Game Logic Handler

