---
description: Architectural reference for MountedUpdate
---

# MountedUpdate

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class MountedUpdate {
```

## Architecture & Concepts
The MountedUpdate class is a specialized Data Transfer Object, not a service or manager. Its sole purpose is to represent a network packet that synchronizes the mounting state of an entity. This includes an entity riding another entity (like a player on a horse) or interacting with a mountable block (like a player in a minecart).

This class is a fundamental building block of the Hytale network protocol layer. It is meticulously designed for high-performance, zero-allocation network I/O. The entire structure is mapped to a **fixed-size 48-byte block**, which simplifies buffer management on both the client and server, preventing network-level fragmentation and unpredictable performance.

A key design feature is the use of a bitmask (`nullBits`) in the first byte of the payload. This is a common bit-packing optimization that allows the protocol to efficiently represent the presence or absence of nullable fields like `attachmentOffset` and `block` without requiring variable-length encoding, thus preserving the fixed-size guarantee.

## Lifecycle & Ownership
- **Creation:**
  - **Sending Peer (e.g., Server):** Instantiated directly using its constructor (`new MountedUpdate(...)`) when a change in an entity's mount state needs to be broadcast. The game state is copied into the new object.
  - **Receiving Peer (e.g., Client):** Instantiated exclusively by the network layer via the static `deserialize` factory method, which reads a 48-byte block from a Netty ByteBuf.

- **Scope:** The lifecycle of a MountedUpdate instance is extremely brief and transient. It exists only for the immediate task of serialization or deserialization. Once its data has been written to a network buffer or read by a game system, it is considered stale and has no further purpose.

- **Destruction:** The object is eligible for garbage collection immediately after being processed by a single system. There is no persistent ownership, and instances must not be cached or reused.

## Internal State & Concurrency
- **State:** The object's state is fully **mutable**, with public fields for direct, low-overhead access. This design choice prioritizes performance over encapsulation, a common and acceptable trade-off for internal, high-throughput DTOs.

- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, serialized, or deserialized within a single thread context (typically a Netty event loop thread or the main game thread).
  - **WARNING:** Sharing a MountedUpdate instance across threads without explicit, external synchronization will result in race conditions and data corruption. Do not write to an instance from one thread while another thread is reading or serializing it.

## API Surface
The public contract is focused entirely on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static MountedUpdate | O(1) | Constructs a new MountedUpdate by reading a fixed 48-byte block from a buffer. |
| serialize(buf) | void | O(1) | Writes the object's state into a fixed 48-byte block in the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object (48). Used for buffer allocation. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough bytes for a valid read. |
| clone() | MountedUpdate | O(1) | Creates a deep copy of the object and its contained value types. |

## Integration Patterns

### Standard Usage
The class is used by network handlers to decode an incoming buffer into a structured object, which is then passed to the appropriate game system.

```java
// Example: In a client-side packet handler
void handlePacket(ByteBuf packetBuffer) {
    // Validate before attempting to read
    if (MountedUpdate.validateStructure(packetBuffer, 0).isError()) {
        // Handle error, disconnect client, etc.
        return;
    }

    // Deserialize the buffer into a usable object
    MountedUpdate update = MountedUpdate.deserialize(packetBuffer, 0);

    // Pass the data to the entity system for processing
    EntitySystem entitySystem = getGameContext().getEntitySystem();
    entitySystem.applyMountUpdate(update);
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching/Pooling:** Do not attempt to pool or reuse MountedUpdate instances. They are lightweight objects, and the overhead of a pooling system is greater than the cost of garbage collection. Reusing instances is a primary source of state corruption bugs.

- **Multi-threaded Access:** Never pass a MountedUpdate instance to another thread for processing without first creating a defensive copy via the `clone()` method.

- **Partial Deserialization:** Do not attempt to read fields from the ByteBuf directly. Always use the `deserialize` method to ensure the `nullBits` field is correctly interpreted and the object is constructed in a valid state.

## Data Pipeline
MountedUpdate serves as the data container that carries mount state from a source of authority (the server) to a replica (the client).

> **Server Flow:**
> Game Logic (Mount Event) -> `new MountedUpdate(...)` -> **MountedUpdate Instance** -> `serialize(buf)` -> Netty Channel

> **Client Flow:**
> Netty Channel -> ByteBuf -> Packet Handler -> `MountedUpdate.deserialize(buf)` -> **MountedUpdate Instance** -> Entity System -> Render State Update

