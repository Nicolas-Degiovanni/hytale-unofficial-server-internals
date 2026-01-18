---
description: Architectural reference for IntParamValue
---

# IntParamValue

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class IntParamValue extends ParamValue {
```

## Architecture & Concepts
IntParamValue is a fundamental Data Transfer Object (DTO) within the Hytale network protocol layer. It serves as a concrete, type-safe representation of a 32-bit signed integer for serialization and deserialization over the network.

As a subclass of the abstract ParamValue, it forms part of a larger system for handling typed parameters within generic packet structures. Its design prioritizes performance and wire-format stability by enforcing a fixed-size, 4-byte block using little-endian byte order. This avoids the processing overhead and potential ambiguity of variable-length integer encodings, making it ideal for frequently transmitted, fixed-range data like entity IDs, health values, or item counts.

The class acts as a low-level data container, bridging the raw byte stream managed by Netty with the higher-level game logic that consumes typed data.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary scenarios:
    1.  By the protocol deserialization pipeline, which invokes the static `deserialize` factory method when an integer parameter is identified in an incoming network buffer.
    2.  By higher-level application or game logic when constructing a new network packet that requires an integer parameter.

- **Scope:** Extremely short-lived. An IntParamValue object is designed to be ephemeral, typically existing only for the duration of a single network packet's processing cycle. Once the primitive integer is extracted or serialized, the object has served its purpose.

- **Destruction:** Managed entirely by the Java Garbage Collector. Due to their transient nature, these objects are prime candidates for rapid collection within the GC's young generation, imposing minimal memory management overhead. **WARNING:** Holding long-term references to IntParamValue objects is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** The class holds a single mutable public field: `value`. While technically mutable, instances are intended to be treated as immutable value objects after their initial construction. The public field is a deliberate performance optimization to eliminate method call overhead for this high-frequency DTO.

- **Thread Safety:** This class is **not thread-safe**. Direct access to the public `value` field from multiple threads without external synchronization will lead to race conditions. This is considered an acceptable trade-off, as network packet processing is typically confined to a single Netty worker thread. Instances must not be shared across threads.

## API Surface
The public API is minimal, focusing exclusively on serialization, deserialization, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static IntParamValue | O(1) | Factory method. Reads 4 little-endian bytes from the buffer at the given offset and constructs a new instance. |
| serialize(ByteBuf) | int | O(1) | Writes the internal integer value as 4 little-endian bytes to the buffer. Returns bytes written. |
| computeSize() | int | O(1) | Returns the constant size of the serialized data, which is always 4. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains at least 4 readable bytes at the offset. |

## Integration Patterns

### Standard Usage
IntParamValue is never used in isolation. It is always created and passed to a higher-level packet builder or extracted from a received packet.

```java
// Example: Constructing a packet with an integer parameter
PlayerHealthPacket packet = new PlayerHealthPacket();
packet.addParameter(new IntParamValue(100));
networkManager.send(packet);

// Example: Reading an integer parameter from a received packet
void onPacketReceived(GenericPacket p) {
    ParamValue param = p.getParameter(0);
    if (param instanceof IntParamValue) {
        int health = ((IntParamValue) param).value;
        // ... process health value
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store IntParamValue instances in caches, game state objects, or other long-lived collections. Extract the primitive `int` value immediately and discard the wrapper object.
- **Cross-Thread Sharing:** Never pass an IntParamValue instance from a network thread to a game logic thread. This is unsafe due to the mutable public field. Instead, pass the primitive `int` value.
- **Manual Byte Manipulation:** Do not attempt to read or write the integer value manually from a ByteBuf. Always use the provided `serialize` and `deserialize` methods to guarantee correct byte order (little-endian) and format.

## Data Pipeline
IntParamValue is a critical component in the data flow between the network stack and game logic.

> **Inbound Flow:**
> Raw TCP Stream -> Netty ByteBuf -> Protocol Deserializer -> **IntParamValue.deserialize()** -> Game Event with primitive int

> **Outbound Flow:**
> Game Logic Command -> **new IntParamValue(int)** -> Protocol Serializer -> Netty ByteBuf -> Raw TCP Stream

