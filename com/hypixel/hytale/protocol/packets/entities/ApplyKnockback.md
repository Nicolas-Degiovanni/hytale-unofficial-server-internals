---
description: Architectural reference for ApplyKnockback
---

# ApplyKnockback

**Package:** com.hypixel.hytale.protocol.packets.entities
**Type:** Transient

## Definition
```java
// Signature
public class ApplyKnockback implements Packet {
```

## Architecture & Concepts
The ApplyKnockback class is a Data Transfer Object (DTO) that represents a specific, atomic command within the Hytale network protocol. Its sole purpose is to instruct a game client that a physics impulse, or "knockback", must be applied to an entity. This packet is a fundamental component of the server-authoritative physics and combat simulation.

As a concrete implementation of the Packet interface, this class is designed for high-performance binary serialization and deserialization. It defines a rigid data structure with a fixed size, ensuring predictable network performance and low-overhead processing. The structure includes a knockback vector (x, y, z), an optional point-of-impact (hitPosition), and a modifier type (changeType) that dictates how the new velocity should be combined with the entity's existing velocity.

This class is not a service or a manager; it is pure data. It acts as a message, flowing from the server's simulation logic, across the network via Netty, to the client's entity processing system.

### Lifecycle & Ownership
-   **Creation:**
    -   **Server-Side:** Instantiated by the server's combat or physics engine when an event (e.g., an explosion, a weapon hit) generates a knockback force on an entity.
    -   **Client-Side:** Instantiated exclusively by the static `deserialize` method when the corresponding network packet (ID 164) is read from the Netty ByteBuf. The network layer owns the initial rehydrated object.

-   **Scope:** Extremely short-lived and ephemeral. An instance exists only for the duration of its journey through the network pipeline and subsequent processing by a packet handler. It does not persist between game ticks.

-   **Destruction:** The object is eligible for garbage collection immediately after the relevant client-side system (e.g., an EntityPhysicsSystem) has consumed its data and applied the velocity change. Ownership is passed from the network handler to the game logic and is never retained.

## Internal State & Concurrency
-   **State:** Highly mutable. All fields are public and directly accessible. The class is a simple container for the knockback data. It contains no internal logic beyond serialization and validation. The state includes the velocity vector, an optional hit position, and the velocity change type.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed within a single thread, typically a Netty network thread or the main game loop thread.
    -   **WARNING:** Concurrent modification of an ApplyKnockback instance from multiple threads will lead to data corruption and unpredictable physics behavior. Do not share instances across threads without explicit, external synchronization.

## API Surface
The public contract is dominated by static methods for protocol handling and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ApplyKnockback | O(1) | Constructs a new ApplyKnockback instance by reading from a ByteBuf at a given offset. This is the primary entry point on the client. |
| serialize(buf) | void | O(1) | Writes the instance's state into the provided ByteBuf. This is the primary exit point on the server. |
| computeSize() | int | O(1) | Returns the fixed size (38 bytes) of the packet on the wire. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a low-level check to ensure the buffer contains enough bytes to read a valid packet. |
| clone() | ApplyKnockback | O(1) | Creates a deep copy of the packet instance. Useful for state buffering or re-queueing. |

## Integration Patterns

### Standard Usage
On the client, this packet is never created directly. It is received by a central packet dispatcher, which then routes it to the appropriate handler for processing. The handler extracts the data and applies it to the game world.

```java
// Executed within a client-side packet handler
void handleApplyKnockback(ApplyKnockback packet) {
    // Assume we have a way to find the target entity
    Entity targetEntity = World.getEntityById(this.targetEntityId);

    if (targetEntity != null) {
        PhysicsComponent physics = targetEntity.getComponent(PhysicsComponent.class);
        Vector3f knockbackVector = new Vector3f(packet.x, packet.y, packet.z);

        // Apply the velocity change as dictated by the server
        physics.applyVelocityChange(knockbackVector, packet.changeType);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Client-Side Instantiation:** Do not use `new ApplyKnockback()` on the client to simulate physics. All physics commands are server-authoritative. Creating this packet on the client will have no effect on the game state.
-   **State Reuse:** Do not modify and re-serialize the same packet instance. This can lead to subtle bugs if a reference is held elsewhere. Always create a new instance for each new knockback event on the server.
-   **Ignoring ChangeType:** The `changeType` field is critical. Failing to interpret it correctly (e.g., always adding velocity instead of setting it) will cause desynchronization between client and server physics.

## Data Pipeline
The flow of ApplyKnockback data is unidirectional from server to client. It represents the replication of a server-side physics event.

> **Server Flow:**
> Combat/Physics Engine -> **ApplyKnockback (new instance)** -> `serialize()` -> Netty Channel -> Network
>
> ---
>
> **Client Flow:**
> Network -> Netty Channel -> `deserialize()` -> **ApplyKnockback (rehydrated instance)** -> Packet Dispatcher -> Entity Physics System -> Entity Velocity Update

