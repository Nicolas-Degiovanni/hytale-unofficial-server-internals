---
description: Architectural reference for ChangeVelocity
---

# ChangeVelocity

**Package:** com.hypixel.hytale.protocol.packets.entities
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ChangeVelocity implements Packet {
```

## Architecture & Concepts
The ChangeVelocity class is a Data Transfer Object (DTO) that represents a discrete network message within the Hytale protocol. Its sole purpose is to encapsulate and transport data required to modify an entity's velocity vector across the network.

As an implementation of the Packet interface, it is a fundamental component of the client-server communication layer. It acts as a structured, serializable container for physics-related state changes. This class is not a service or manager; it is the raw data that flows through the network and entity synchronization systems. Its design prioritizes performance and a minimal network footprint, evidenced by its fixed-size structure and use of a bitmask for optional fields.

This packet is critical for synchronizing entity physics, enabling effects like knockback, projectiles, and character controller impulses to be replicated from the server to the client.

## Lifecycle & Ownership
- **Creation:** An instance is created on the sending side (typically the server) when game logic dictates a change in an entity's velocity. On the receiving side, a new instance is created by the static deserialize method when the network layer decodes an incoming byte stream with a packet ID of 163.
- **Scope:** The lifecycle of a ChangeVelocity object is extremely brief and ephemeral. It exists only for the duration of a single network transaction: serialization, transmission, deserialization, and processing.
- **Destruction:** The object is eligible for garbage collection immediately after its data has been consumed by the relevant system, such as an entity manager or physics engine. It is not retained or reused.

## Internal State & Concurrency
- **State:** The internal state is fully mutable via public fields. This design choice is common in high-performance DTOs to avoid the overhead of getters, setters, and builder patterns during the critical serialization/deserialization process. The object holds no cached data and its state directly represents the on-the-wire format.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and processed within a single thread, such as a Netty event loop thread or the main game thread. Concurrent modification from multiple threads will lead to data corruption and unpredictable behavior. All synchronization must be handled externally.

## API Surface
The public contract is defined by its fields and its serialization/deserialization logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ChangeVelocity | O(1) | Constructs a new ChangeVelocity instance from a raw ByteBuf. Critical for network ingress. |
| serialize(buf) | void | O(1) | Writes the object's state into the provided ByteBuf. Critical for network egress. |
| computeSize() | int | O(1) | Returns the fixed size of the packet in bytes (35). Used for buffer allocation. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a fast, preliminary check to ensure a buffer is large enough to contain the packet. |

## Integration Patterns

### Standard Usage
This packet is intended to be used exclusively by the core networking and entity systems. Game logic on the server creates and populates the packet, which is then passed to a network channel for dispatch. The client-side network handler receives the bytes, decodes them into a ChangeVelocity object, and applies the update to the corresponding local entity.

```java
// Example: Server-side logic creating and sending the packet
// NOTE: This is a conceptual example. Direct channel writing is handled by the network engine.

ChangeVelocity velocityUpdate = new ChangeVelocity(
    10.5f, 0.0f, -5.0f, ChangeVelocityType.Set, null
);

// The network engine would then serialize and send this packet to the client.
networkChannel.writeAndFlush(velocityUpdate);
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not retain and modify a ChangeVelocity instance to send multiple updates. Its mutable nature makes this pattern highly error-prone. Always create a new instance for each distinct message.
- **Manual Serialization:** Avoid calling serialize or deserialize directly in application-level game logic. These methods are low-level and should only be invoked by the protocol's encoding and decoding pipeline.
- **Cross-Thread Access:** Never share a ChangeVelocity instance across threads without explicit and robust external locking. The object is not designed for concurrent access.

## Data Pipeline
The ChangeVelocity packet is a data container that moves through the network stack. Its flow is linear and unidirectional for a single transaction.

> **Outbound Flow (e.g., Server):**
> Physics Engine -> Game Logic -> **new ChangeVelocity()** -> Network Encoder (serialize) -> TCP/UDP Socket

> **Inbound Flow (e.g., Client):**
> TCP/UDP Socket -> Packet Decoder (deserialize) -> **ChangeVelocity instance** -> Entity System -> Local Physics Update

