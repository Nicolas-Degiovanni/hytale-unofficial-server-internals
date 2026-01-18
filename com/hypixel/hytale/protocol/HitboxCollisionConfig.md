---
description: Architectural reference for HitboxCollisionConfig
---

# HitboxCollisionConfig

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class HitboxCollisionConfig {
```

## Architecture & Concepts
The HitboxCollisionConfig class is a specialized Plain Old Java Object (POJO) designed for high-performance network serialization. It functions as a fixed-size data structure, or *struct*, within the Hytale protocol layer, explicitly defining the physical collision properties of an entity's hitbox.

Its primary architectural role is to serve as a direct, 1-to-1 mapping of a 5-byte block within a network packet's payload. The class design prioritizes serialization speed and memory predictability over complex behavior. All operations, including serialization, deserialization, and validation, are implemented as static utility functions or simple instance methods that operate directly on Netty ByteBuf objects. This avoids intermediate object allocations and minimizes processing overhead on the network I/O threads.

This class is not a service or manager; it is a pure data container that represents a contract between the client and server on how collision data is structured in transit.

### Lifecycle & Ownership
- **Creation:** An instance is created under two primary conditions:
    1. **Deserialization:** The static `deserialize` method is invoked by a higher-level protocol decoder when parsing an incoming network packet. This is the most common creation path.
    2. **Manual Instantiation:** Game logic instantiates this class to configure an entity's properties before that entity is serialized and sent over the network.
- **Scope:** The object's lifetime is extremely short. It typically exists only within the scope of a single method call for packet processing or entity creation. It is not intended to be stored long-term.
- **Destruction:** The object is managed by the Java Garbage Collector and is eligible for cleanup as soon as it falls out of scope. No manual resource management is required.

## Internal State & Concurrency
- **State:** The state of HitboxCollisionConfig is **mutable**. Its public fields, `collisionType` and `softCollisionOffsetRatio`, can be modified after instantiation. The object's in-memory representation is always a fixed 5 bytes, as defined by the `FIXED_BLOCK_SIZE` constant.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. It is designed to be confined to a single thread, such as a Netty I/O worker or the main game thread.

    **WARNING:** Sharing an instance of HitboxCollisionConfig across multiple threads without external synchronization will lead to race conditions and unpredictable behavior. Do not store instances in shared collections or pass them between threads without ensuring exclusive access.

## API Surface
The public API is minimal, focusing exclusively on serialization and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | HitboxCollisionConfig | O(1) | **Static Factory.** Reads 5 bytes from the buffer at the given offset and constructs a new instance. |
| serialize(ByteBuf) | void | O(1) | Writes the object's 5-byte state into the provided buffer. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Static Utility.** Performs a bounds check to ensure the buffer contains at least 5 readable bytes from the offset. |
| computeSize() | int | O(1) | Returns the constant size of the serialized data, which is always 5. |
| clone() | HitboxCollisionConfig | O(1) | Creates a shallow copy of the object. |

## Integration Patterns

### Standard Usage
This class is almost never used in isolation. It is typically a component of a larger network packet or entity definition. Its static methods are called during the parent object's serialization lifecycle.

```java
// Example: Deserializing as part of a larger packet
public class EntityDefinitionPacket {
    // ... other fields
    private HitboxCollisionConfig collisionConfig;

    public void deserialize(ByteBuf buffer) {
        // ... deserialize other fields
        int currentOffset = ...;
        ValidationResult result = HitboxCollisionConfig.validateStructure(buffer, currentOffset);
        if (!result.isOk()) {
            throw new ProtocolException(result.getErrorMessage());
        }
        this.collisionConfig = HitboxCollisionConfig.deserialize(buffer, currentOffset);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not reuse a single instance for multiple entities or packets. The mutable state can easily lead to data corruption. Always create a new instance or clone an existing template.
- **Concurrent Access:** Never modify an instance from one thread while it is being read or serialized by another. This will corrupt the network buffer.
- **Ignoring Validation:** Bypassing the `validateStructure` check before calling `deserialize` can lead to buffer under-read exceptions and server instability if a malformed packet is received.

## Data Pipeline
The flow of data through this component is bidirectional, representing its role in both reading and writing network traffic.

> **Ingress Flow (Client/Server Receiving Data):**
> Raw ByteBuf -> Protocol Decoder -> **HitboxCollisionConfig.deserialize** -> Game Entity Property

> **Egress Flow (Client/Server Sending Data):**
> Game Entity Property -> **new HitboxCollisionConfig()** -> Protocol Encoder -> **HitboxCollisionConfig.serialize** -> Raw ByteBuf

