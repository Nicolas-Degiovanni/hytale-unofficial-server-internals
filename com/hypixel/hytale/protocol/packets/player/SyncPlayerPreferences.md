---
description: Architectural reference for SyncPlayerPreferences
---

# SyncPlayerPreferences

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Transient

## Definition
```java
// Signature
public class SyncPlayerPreferences implements Packet {
```

## Architecture & Concepts
SyncPlayerPreferences is a Data Transfer Object (DTO) that represents a network message within the Hytale protocol. Its sole purpose is to transmit a player's client-side settings and preferences between the client and the server, ensuring a consistent gameplay experience.

As an implementation of the Packet interface, this class is a fundamental component of the low-level network layer. It is designed for extreme performance and predictability. The structure is defined by a fixed-size data block of 8 bytes, with no variable-length fields. This design choice eliminates the need for complex size calculations or parsing loops, allowing for near-instantaneous serialization and deserialization directly from a network buffer.

This packet acts as a data contract, ensuring that both the client and server agree on the exact binary layout for player preference information. It does not contain any game logic; it is a pure data container that is created, transmitted, and consumed by other systems.

### Lifecycle & Ownership
- **Creation:** An instance is created under two primary conditions:
    1. **Inbound:** The network protocol dispatcher calls the static `deserialize` method when an incoming network buffer with Packet ID 116 is detected.
    2. **Outbound:** Game logic on the sending endpoint (client or server) instantiates a new SyncPlayerPreferences object, populates its fields with the current player state, and queues it for transmission.
- **Scope:** The object's lifetime is exceptionally short. It exists only for the brief moment it is being processed by a network handler or being serialized into a buffer. It is a fire-and-forget data structure.
- **Destruction:** The object is managed by the Java Garbage Collector and is eligible for cleanup as soon as it falls out of the scope of the network handler or game logic that created it. There are no native resources or manual cleanup procedures required.

## Internal State & Concurrency
- **State:** The internal state is fully mutable via public fields. This design prioritizes raw performance for serialization over encapsulation. Once deserialized from a network buffer or populated for sending, an instance is typically treated as immutable by convention.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data object with no internal locking or synchronization. It is designed to be created, populated, and read within a single thread, such as a Netty I/O worker or the main game thread.

**WARNING:** Sharing an instance of SyncPlayerPreferences across multiple threads without explicit external synchronization will lead to race conditions and unpredictable behavior.

## API Surface
The public API is minimal, focusing entirely on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | SyncPlayerPreferences | O(1) | **Static Factory.** Constructs a new packet by reading 8 bytes from the buffer. |
| serialize(ByteBuf) | void | O(1) | Writes the packet's 8-byte state into the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the packet, which is always 8. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static.** Checks if the buffer has enough readable bytes for a valid packet. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (116). |

## Integration Patterns

### Standard Usage
The primary integration point is within a network channel handler or a packet dispatcher. The handler reads the packet ID, validates the buffer, and uses the static deserialize method to create the object. The object is then passed to a higher-level system for processing.

```java
// In a network handler receiving a ByteBuf
if (packetId == SyncPlayerPreferences.PACKET_ID) {
    ValidationResult result = SyncPlayerPreferences.validateStructure(buffer, offset);
    if (result.isOk()) {
        SyncPlayerPreferences prefs = SyncPlayerPreferences.deserialize(buffer, offset);
        // Pass the data object to the player's session manager
        playerSession.applyPreferences(prefs);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify a SyncPlayerPreferences object after it has been queued for serialization. The network layer may read its state from a different thread, leading to data corruption. Always create a new instance for each message.
- **Cross-Thread Sharing:** Do not deserialize a packet on a network thread and pass the same instance to multiple game logic threads without a deep copy or proper synchronization. Its mutable nature makes it inherently unsafe for concurrent access.
- **Manual Serialization:** Do not manually write the fields to a buffer. Always use the provided `serialize` method to ensure the byte order and layout conform to the protocol specification.

## Data Pipeline
The flow for an inbound packet is linear and designed for high throughput. The SyncPlayerPreferences class is a critical link where raw bytes are transformed into a structured, usable data object.

> Flow:
> Network ByteBuf -> Protocol Dispatcher -> **SyncPlayerPreferences.deserialize** -> Player Session Handler -> Player State Update
---

