---
description: Architectural reference for TargetedDamage
---

# TargetedDamage

**Package:** com.hypixel.hytale.protocol
**Type:** Packet Struct / DTO

## Definition
```java
// Signature
public class TargetedDamage {
```

## Architecture & Concepts
The TargetedDamage class is a data transfer object (DTO) that represents a single, discrete damage event within the Hytale network protocol. It is not a service or manager, but rather a structured container for data transmitted between the client and server.

Its design is highly optimized for network efficiency. The class implements a custom serialization and deserialization scheme that operates directly on Netty ByteBufs. This avoids the overhead of intermediate representations like JSON or reflection-based serializers.

Key architectural features include:
*   **Fixed and Variable Sizing:** The serialized structure consists of a 9-byte fixed-size block for core fields and a variable-sized block for optional, complex data like DamageEffects.
*   **Nullable Bit Field:** A single byte acts as a bitmask to indicate the presence or absence of nullable fields. This is a common pattern in performance-critical protocols to avoid sending unnecessary data, minimizing packet size.
*   **Linked-List Structure:** The fields `index` and `next` suggest this object is designed to be part of a logical linked list of damage events, even though they are serialized into a contiguous buffer. This allows the protocol to represent complex combat sequences involving multiple targets or damage ticks.

This class is a fundamental component of the combat data model, acting as the on-the-wire representation of damage applied to a specific entity.

## Lifecycle & Ownership
-   **Creation:** TargetedDamage instances are created under two primary circumstances:
    1.  **Inbound:** The static `deserialize` method is called by a higher-level protocol decoder when parsing an incoming network packet from a ByteBuf.
    2.  **Outbound:** Game logic on the sending side (typically the server) instantiates a new TargetedDamage object and populates its fields before it is passed to a serializer.
-   **Scope:** The object's lifetime is extremely short and tied to the scope of a single network packet's processing cycle. It is a frame-transient object.
-   **Destruction:** Instances are not managed by any container or pool. They become eligible for garbage collection as soon as the parent network packet has been fully processed by the game logic or written to the network socket. Holding long-term references to these objects is a memory leak anti-pattern.

## Internal State & Concurrency
-   **State:** The class is fully mutable, with public fields intended for direct access. It is a simple data holder and contains no internal caching, logic, or side effects. Its state is a direct representation of the data read from or written to the network buffer.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and read by a single thread, typically a Netty I/O worker thread or the main game thread. Concurrent modification from multiple threads will result in data corruption and is strictly unsupported. All synchronization must be handled externally by the calling system.

## API Surface
The public API is focused exclusively on serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static TargetedDamage | O(N) | Constructs a new TargetedDamage instance by reading from a ByteBuf at a given offset. N is the size of the variable data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf using the custom protocol format. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. Crucial for pre-allocating buffers. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a structural validation on the data in a ByteBuf without the overhead of full object instantiation. |
| clone() | TargetedDamage | O(N) | Creates a deep copy of the object and its contained data structures. |

## Integration Patterns

### Standard Usage
The primary interaction pattern is deserialization from a network buffer during packet handling. The object is then used to pass combat data to the relevant game systems.

```java
// Within a packet decoder or game event handler
void handleCombatData(ByteBuf packetData) {
    // Validate there is enough data to be a valid TargetedDamage
    ValidationResult result = TargetedDamage.validateStructure(packetData, 0);
    if (!result.isValid()) {
        // Handle malformed packet
        return;
    }

    TargetedDamage damageEvent = TargetedDamage.deserialize(packetData, 0);

    // Pass the structured data to the game simulation
    gameWorld.applyDamage(damageEvent);
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not store instances of TargetedDamage in caches, game state components, or any structure that outlives a single frame or packet processing cycle. This will cause memory leaks.
-   **Cross-Thread Modification:** Do not read a TargetedDamage on one thread while another thread is deserializing or modifying it. This will lead to race conditions and inconsistent state.
-   **Instance Re-use:** Do not attempt to pool or re-use TargetedDamage objects. They are lightweight and designed to be created and discarded rapidly. Re-using them can lead to subtle bugs from stale data.

## Data Pipeline
TargetedDamage serves as the structured data format within the network protocol pipeline. It is the output of a decoding step and the input to game logic.

> Flow:
> Network ByteBuf -> Protocol Packet Decoder -> **TargetedDamage.deserialize()** -> TargetedDamage Instance -> Game Logic / Combat System

