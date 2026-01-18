---
description: Architectural reference for EasingConfig
---

# EasingConfig

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class EasingConfig {
```

## Architecture & Concepts
The EasingConfig class is a low-level, fixed-size data structure used exclusively within the Hytale network protocol layer. It is not a service or a manager, but rather a fundamental data container that represents the configuration for a time-based interpolation function, commonly known as an "easing" function.

Its primary role is to define the precise binary layout for transmitting animation, movement, or transition parameters between the client and server. The class encapsulates the serialization and deserialization logic required to convert its state to and from a Netty ByteBuf, ensuring a consistent and compact network representation.

Architecturally, EasingConfig acts as a "message component" or a "packet field". It is almost never used in isolation but is instead composed within larger, more complex protocol message objects. Its design prioritizes performance and network efficiency over flexibility, as evidenced by its fixed 5-byte size.

## Lifecycle & Ownership
-   **Creation:** EasingConfig instances are created under two primary scenarios:
    1.  **Deserialization:** The static factory method `deserialize` is invoked by a higher-level packet parser when reading an incoming network buffer. This is the most common creation path.
    2.  **Manual Instantiation:** Game logic constructs a new EasingConfig instance when preparing data to be sent over the network. This typically occurs before a call to a serializer for a larger packet.

-   **Scope:** Instances are **transient and short-lived**. An EasingConfig object typically exists only for the duration of processing a single network packet or constructing an outgoing one. It is not intended for long-term storage.

-   **Destruction:** The object is managed by the Java Garbage Collector and is eligible for cleanup as soon as it falls out of the scope of the packet processing logic. No manual destruction or resource cleanup is required.

## Internal State & Concurrency
-   **State:** The internal state is **mutable**. The public fields `time` and `type` can be modified directly after instantiation. The class is a simple data holder and performs no caching or complex state management.

-   **Thread Safety:** This class is **not thread-safe**. It contains no synchronization primitives. It is designed to be created, populated, and read within a single-threaded context, such as a Netty I/O worker thread or the main game logic thread.

    **Warning:** Sharing an EasingConfig instance across multiple threads is a severe anti-pattern and will lead to data corruption and race conditions. Do not pass instances between threads without ensuring proper synchronization or creating a defensive copy.

## API Surface
The public contract is focused on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EasingConfig() | constructor | O(1) | Creates a default instance. |
| EasingConfig(float, EasingType) | constructor | O(1) | Creates a fully initialized instance. |
| deserialize(ByteBuf, int) | static EasingConfig | O(1) | Constructs an instance by reading 5 bytes from a buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the instance's 5 bytes of state into the provided buffer. |
| computeSize() | int | O(1) | Returns the fixed network size of the object, which is always 5. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if a buffer contains enough readable bytes to deserialize an instance. |

## Integration Patterns

### Standard Usage
EasingConfig is intended to be used as a component of a larger network message. It should be serialized into or deserialized from a parent object's buffer.

```java
// Example: Deserializing from a parent packet's buffer
public void deserialize(ByteBuf parentBuffer) {
    // ... read other parent packet fields ...
    int easingConfigOffset = ...; // Calculate offset within the parent buffer
    this.animationEasing = EasingConfig.deserialize(parentBuffer, easingConfigOffset);
    // ... continue reading ...
}

// Example: Serializing into a parent packet's buffer
public void serialize(ByteBuf parentBuffer) {
    // ... write other parent packet fields ...
    EasingConfig config = new EasingConfig(1.5f, EasingType.EaseInOutQuad);
    config.serialize(parentBuffer);
    // ... continue writing ...
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term State:** Do not hold references to EasingConfig objects in long-lived components or caches. Its mutable nature makes it unsuitable for this purpose. If you need to store the data, copy the `time` and `type` values into a more stable, domain-specific object.
-   **Cross-Thread Sharing:** Never modify an EasingConfig instance from a different thread than the one that created it without external locking. This will cause unpredictable behavior.
-   **Manual Size Calculation:** Do not hardcode the size `5` in your application logic. Always use the `FIXED_BLOCK_SIZE` constant or the `computeSize` method to ensure correctness if the protocol ever changes.

## Data Pipeline
EasingConfig is a low-level participant in the network data pipeline. It does not initiate data flow but is acted upon by higher-level serializers and deserializers.

> **Ingress Flow (Receiving Data):**
> Netty Channel -> ByteBuf -> Parent Packet Deserializer -> **EasingConfig.deserialize()** -> Game Logic

> **Egress Flow (Sending Data):**
> Game Logic -> new EasingConfig(...) -> Parent Packet Serializer -> **easingConfig.serialize()** -> ByteBuf -> Netty Channel

