---
description: Architectural reference for VelocityConfig
---

# VelocityConfig

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (Transient)

## Definition
```java
// Signature
public class VelocityConfig {
```

## Architecture & Concepts
The VelocityConfig class is a fundamental component of the Hytale network protocol layer. It is not a service or manager, but rather a **protocol message struct** designed for high-performance data exchange. Its primary responsibility is to represent a fixed-size, 21-byte block of data defining an entity's physical velocity and resistance properties.

Architecturally, this class acts as a direct, low-level mapping between the in-memory game state and the on-the-wire byte representation. The design intentionally favors performance and simplicity over encapsulation; public fields allow for direct, high-speed access by the physics engine and the network serialization pipeline. This pattern is common in performance-critical systems where the overhead of getters and setters is undesirable and the object's role is purely to carry data.

## Lifecycle & Ownership
- **Creation:** VelocityConfig instances are created under two primary scenarios:
    1.  **Deserialization:** The static factory method *deserialize* is invoked by the network protocol decoder when a corresponding packet is read from a Netty ByteBuf.
    2.  **Manual Instantiation:** Game logic, such as the physics engine or entity configuration systems, creates instances to populate with state that needs to be transmitted over the network.
- **Scope:** This is a **short-lived, transient object**. Its lifetime is strictly bound to the scope of a single network packet's processing or a single game state update. It is not designed for long-term storage or caching.
- **Destruction:** Instances are managed by the Java Garbage Collector. They become eligible for collection as soon as they are no longer referenced, which typically occurs immediately after their data has been applied to a game entity or serialized into an outbound network buffer.

## Internal State & Concurrency
- **State:** The object is **fully mutable**. All data-holding fields are public, allowing any consumer to modify its state after creation. This design choice prioritizes performance but requires disciplined usage.
- **Thread Safety:** This class is **not thread-safe**. Concurrent modification and access from multiple threads will result in race conditions, memory visibility issues, and data corruption. It is designed to be confined to a single thread, such as a Netty I/O worker thread or the main game loop thread.

**WARNING:** Never share a VelocityConfig instance across threads without explicit and robust synchronization mechanisms, which would negate the performance benefits of its design.

## API Surface
The public API is centered on serialization and validation for network transport.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static VelocityConfig | O(1) | Constructs a new VelocityConfig by reading 21 bytes from a buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state as 21 bytes into the provided buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a bounds check to ensure at least 21 bytes are readable from the buffer. |
| clone() | VelocityConfig | O(1) | Creates a new instance with an identical copy of the data. |

## Integration Patterns

### Standard Usage
The typical use case involves a full serialization-deserialization round trip. The sending system populates the object and serializes it, while the receiving system validates the buffer and deserializes it.

```java
// Example: Server preparing and sending data
VelocityConfig configToSend = new VelocityConfig(0.9f, 0.95f, 0.98f, 0.99f, 5.0f, VelocityThresholdStyle.Linear);
ByteBuf networkBuffer = Unpooled.buffer();
configToSend.serialize(networkBuffer);
// ... write buffer to network channel

// Example: Client receiving and processing data
// 'receivedBuffer' is a ByteBuf from the network
if (VelocityConfig.validateStructure(receivedBuffer, 0).isOk()) {
    VelocityConfig receivedConfig = VelocityConfig.deserialize(receivedBuffer, 0);
    // Apply receivedConfig to a local player entity's physics component
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term State Management:** Do not retain instances of VelocityConfig as part of a long-term component state. They represent a point-in-time snapshot and should be used to update a more permanent state object, then discarded.
- **Cross-Thread Sharing:** Do not pass an instance to another thread for processing. This is a critical error. If data must be shared, serialize it to a buffer or create a deep copy (using the clone method or copy constructor) for the other thread.
- **Ignoring Validation:** Never call *deserialize* on a network buffer without first calling *validateStructure*. Failing to do so can result in an IndexOutOfBoundsException, which may be used as a vector for denial-of-service attacks.

## Data Pipeline
VelocityConfig serves as a data payload within the network protocol. It does not process data itself but is the data being processed.

> **Outbound Flow:**
> Game Physics State -> **VelocityConfig** (Instantiation) -> serialize() -> Netty ByteBuf -> Network Socket

> **Inbound Flow:**
> Network Socket -> Netty ByteBuf -> validateStructure() -> deserialize() -> **VelocityConfig** (Instance) -> Game Physics Update

