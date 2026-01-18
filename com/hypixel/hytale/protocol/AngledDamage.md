---
description: Architectural reference for AngledDamage
---

# AngledDamage

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class AngledDamage {
```

## Architecture & Concepts
AngledDamage is a fundamental data structure within the Hytale network protocol layer. It is not a service or manager, but rather a Plain Old Java Object (POJO) that serves as a Data Transfer Object (DTO) for serializing damage events that have a directional component.

Its design is heavily optimized for high-performance network I/O, prioritizing low allocation overhead and direct binary manipulation. The class defines a strict binary layout for damage information, consisting of a fixed-size block for core data and a variable-size block for optional, nested structures like DamageEffects.

The API is dominated by static methods that operate directly on Netty ByteBuf instances. This pattern allows the network layer to read, validate, and skip over data segments without the overhead of instantiating an AngledDamage object, which is critical for processing large streams of game events efficiently. The presence of a `next` field suggests this structure can be used to form an intrusive linked list within a contiguous buffer, a common technique for serializing collections without extra allocation.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1.  **Deserialization:** The static `deserialize` method is called by a network packet decoder (e.g., a Netty ChannelInboundHandler) when an incoming data stream is processed.
    2.  **Serialization:** Game logic on the server instantiates and populates an AngledDamage object before passing it to a serializer to be written into an outgoing network packet.

- **Scope:** Extremely short-lived and transient. An instance is intended to exist only for the duration of a single network event processing cycle or a single game tick.

- **Destruction:** The object is managed by the Java Garbage Collector. Due to its transient nature, it becomes eligible for collection almost immediately after its data has been consumed by the relevant game system. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The state of AngledDamage is fully mutable and exposed via public fields. It is a simple data container and holds no caches, locks, or other complex internal state. Its structure is defined by the network protocol and directly maps to a binary representation.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and read within a single, well-defined thread, such as a Netty I/O worker thread or the main game logic thread. Sharing instances across threads without explicit, external synchronization will result in race conditions and data corruption.

## API Surface
The public contract is focused on serialization and deserialization from a raw byte buffer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static AngledDamage | O(N) | Constructs an AngledDamage instance by reading from a buffer at a given offset. N is the size of the nested DamageEffects. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided buffer according to the protocol's binary layout. |
| computeSize() | int | O(N) | Calculates the total number of bytes this specific instance will occupy when serialized. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the size of a serialized AngledDamage object directly from a buffer without creating an instance. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a structural integrity check on the data in a buffer without full deserialization. Crucial for protocol security and stability. |
| clone() | AngledDamage | O(N) | Creates a deep copy of the object, including a clone of the nested DamageEffects if present. |

## Integration Patterns

### Standard Usage
AngledDamage is almost exclusively used within the network layer or by game systems preparing data for network transmission. A typical client-side use case involves deserializing it from a packet buffer and passing the data to a game world system.

```java
// Example from within a network packet handler
public void handlePacket(ByteBuf packetBuffer) {
    // Assume 'offset' points to the start of the AngledDamage data
    if (AngledDamage.validateStructure(packetBuffer, offset).isValid()) {
        AngledDamage damageInfo = AngledDamage.deserialize(packetBuffer, offset);
        
        // Pass the structured data to the appropriate game system
        gameWorld.applyDirectionalDamage(targetEntity, damageInfo);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Retention:** Do not store references to AngledDamage instances across multiple game ticks or events. They represent a point-in-time snapshot and should be considered immutable after creation, even though their fields are public.

- **Cross-Thread Sharing:** Never pass an AngledDamage instance between threads. If data needs to be shared, extract the primitive values into a thread-safe structure.

- **Manual Serialization:** Avoid writing the fields to a buffer manually. Always use the `serialize` method to ensure compliance with the binary protocol, especially the handling of the nullable bitfield.

## Data Pipeline
AngledDamage acts as a data contract for transmitting directional damage events. The flow differs slightly between the server (origin) and client (consumer).

**Client-Side Data Ingress:**
> Flow:
> Raw TCP Packet -> Netty ByteBuf -> Protocol Decoder -> **AngledDamage.deserialize** -> Game World System -> Entity Health Update

