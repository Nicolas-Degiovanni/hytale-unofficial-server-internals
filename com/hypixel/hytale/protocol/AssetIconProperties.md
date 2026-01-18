---
description: Architectural reference for AssetIconProperties
---

# AssetIconProperties

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Object

## Definition
```java
// Signature
public class AssetIconProperties {
```

## Architecture & Concepts
AssetIconProperties is a specialized Data Transfer Object (DTO) designed for high-performance network serialization and deserialization. It is not a service or a manager, but a fundamental data structure representing the visual transformation properties—scale, translation, and rotation—of an in-game asset icon.

Its primary architectural role is to define a strict, fixed-size binary layout for this data. The class eschews reflection-based serialization in favor of direct, manual manipulation of Netty's ByteBuf. This design choice prioritizes raw performance and memory predictability, which are critical in the network protocol layer.

The public static constants, such as FIXED_BLOCK_SIZE and NULLABLE_BIT_FIELD_SIZE, expose the underlying binary contract, making the class a self-documenting specification for a 25-byte data block within a network packet.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1.  By the protocol layer via the static `deserialize` factory method when decoding an incoming network packet.
    2.  By game logic via its constructors when preparing icon data to be sent over the network.
- **Scope:** The lifetime of an AssetIconProperties instance is expected to be extremely short. It is a transient object, typically scoped to the processing of a single network packet or a single game state update. It is not designed to be stored long-term.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as all references to it are dropped, which usually occurs immediately after its data has been consumed by game logic or written to an outbound buffer.

## Internal State & Concurrency
- **State:** The object's state is **mutable**. Its core fields—scale, translation, and rotation—are public and can be modified after instantiation. It does not maintain any internal cache or derived state. The state directly represents the data to be serialized.

- **Thread Safety:** This class is **not thread-safe**. All fields are accessed directly without any synchronization primitives.

    **WARNING:** Sharing an instance of AssetIconProperties across multiple threads without external locking mechanisms is unsafe and will lead to data corruption and race conditions. It is designed for use within a single-threaded context, such as a Netty I/O thread or the main game update loop.

## API Surface
The public API is focused exclusively on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static AssetIconProperties | O(1) | Constructs an object by reading a fixed 25-byte block from a ByteBuf. |
| serialize(buf) | void | O(1) | Writes the object's state into a fixed 25-byte block in the provided ByteBuf. |
| computeSize() | int | O(1) | Returns the constant size of the binary representation, which is always 25 bytes. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a pre-flight check to ensure a buffer contains enough bytes for a valid read. |
| clone() | AssetIconProperties | O(1) | Creates a deep copy of the object, including its Vector fields. |

## Integration Patterns

### Standard Usage
AssetIconProperties is almost always used as part of a larger data serialization or deserialization process.

```java
// Example: Deserializing from a network buffer
ValidationResult result = AssetIconProperties.validateStructure(networkBuffer, offset);
if (result.isOk()) {
    AssetIconProperties props = AssetIconProperties.deserialize(networkBuffer, offset);
    // Use the deserialized properties in game logic
    applyIconTransforms(props.scale, props.translation, props.rotation);
}

// Example: Serializing to an outbound buffer
Vector2f newTranslation = new Vector2f(10.0f, -5.0f);
AssetIconProperties newProps = new AssetIconProperties(1.5f, newTranslation, null);
newProps.serialize(outboundBuffer);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not deserialize data into a pre-existing AssetIconProperties instance. This can leave stale data if nullable fields are not correctly overwritten. Always use the object returned by the static `deserialize` method.
- **Cross-Thread Sharing:** Do not pass an instance from a network thread to a game logic thread without first creating a deep copy using the `clone` method. Direct sharing will introduce severe concurrency bugs.
- **Ignoring Validation:** Never call `deserialize` on an untrusted or dynamic data stream without first calling `validateStructure`. Failure to do so can result in an IndexOutOfBoundsException and crash the network pipeline.

## Data Pipeline
AssetIconProperties serves as the in-memory representation of a fixed-format data block. It is a critical link in the chain between raw network bytes and structured game state.

> **Inbound Flow:**
> Network Packet (ByteBuf) → `AssetIconProperties.deserialize` → **AssetIconProperties Instance** → Game Logic / State Update

> **Outbound Flow:**
> Game Logic / State → **AssetIconProperties Instance** → `AssetIconProperties.serialize` → Network Packet (ByteBuf)

