---
description: Architectural reference for InteractionCooldown
---

# InteractionCooldown

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class InteractionCooldown {
```

## Architecture & Concepts

The InteractionCooldown class is a specialized Data Transfer Object (DTO) designed for high-performance network communication. It is not a service or manager, but rather a passive data structure that represents the state of an entity's interaction cooldowns. Its primary role is to define the precise binary layout—the wire format—for this information when sent between the client and server.

The architecture of its serialization format is optimized for speed and minimal payload size, avoiding the overhead of generic serialization frameworks. It employs a hybrid fixed-size and variable-size block strategy:

1.  **Null Bit Field:** The first byte of the serialized data is a bitmask. Each bit corresponds to a nullable, variable-length field (like cooldownId or chargeTimes), indicating whether its data is present in the payload. This avoids wasting space for null values.
2.  **Fixed-Size Block:** A contiguous block of 8 bytes follows the null bit field. It contains fixed-size primitive types like floats and booleans. This allows for extremely fast, direct-offset reads without any parsing.
3.  **Offset Pointers:** The fixed-size block is followed by a series of 4-byte integer offsets. Each offset points to the start of a corresponding variable-length field's data within the variable block. This indirection allows the decoder to jump directly to any field without reading the preceding ones.
4.  **Variable-Size Block:** The final section of the payload contains the actual data for variable-length fields, such as the UTF-8 bytes for cooldownId and the list of floats for chargeTimes.

This structure is a common pattern in high-throughput game networking, prioritizing deserialization performance and bandwidth efficiency over ease of use or flexibility.

## Lifecycle & Ownership

-   **Creation:** Instances are created under two circumstances:
    1.  **Inbound:** The network protocol layer instantiates the object via the static `deserialize` method when decoding an incoming packet from a Netty ByteBuf. The game logic receives a fully-formed object.
    2.  **Outbound:** The game logic instantiates the object using its constructor (`new InteractionCooldown(...)`) to define a cooldown state that needs to be sent over the network.
-   **Scope:** The object is extremely short-lived and transient. Its lifetime is typically confined to the scope of a single network packet processing cycle. It is created, its data is read or written, and it immediately becomes eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual resource management or `close` methods.

## Internal State & Concurrency

-   **State:** The class is a mutable data container. All of its fields are public, allowing for direct, low-overhead modification. This design choice prioritizes performance within the single-threaded context of the network or game loop. It holds no caches or derived state.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and read within a single thread, such as a Netty I/O worker thread or the main game tick thread. Sharing an instance across threads without explicit, external synchronization will lead to memory visibility issues and race conditions. Do not pass instances of this class between threads.

## API Surface

The public API is primarily concerned with serialization, deserialization, and validation of the network wire format.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | InteractionCooldown | O(N) | **[Deserializer]** Constructs an object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | **[Serializer]** Writes the object's state into the provided ByteBuf according to the defined wire format. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a non-allocating check on a buffer to verify if it contains a structurally valid object. Crucial for security and preventing crashes from malformed packets. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will consume when serialized. Useful for pre-allocating buffers. |
| computeBytesConsumed(ByteBuf, int) | int | O(N) | Reads just enough of a buffer to determine the total size of a serialized object without full deserialization. |

*Complexity O(N) refers to the total length of variable-size fields (e.g., string length, array size).*

## Integration Patterns

### Standard Usage

A developer typically creates an instance to send data. The network layer handles receiving and deserializing.

```java
// Example: Sending a new cooldown state to a client
InteractionCooldown newCooldown = new InteractionCooldown(
    "bow_draw",
    1.5f,
    false,
    new float[]{0.25f, 0.75f, 1.25f},
    false,
    true
);

// The object is then attached to a larger packet and sent.
// The framework calls newCooldown.serialize() internally.
somePacket.setCooldownData(newCooldown);
networkManager.sendPacket(somePacket);
```

### Anti-Patterns (Do NOT do this)

-   **Stateful Reuse:** Do not reuse an InteractionCooldown instance across multiple packets or game ticks. They are cheap to create and should be treated as disposable. Reusing an object can lead to bugs where stale data is accidentally sent.
-   **Manual Deserialization:** Game logic should never call `deserialize` directly. The protocol decoding pipeline is responsible for this. Your code should expect to receive a fully constructed object as a method parameter.
-   **Cross-Thread Modification:** Never modify an InteractionCooldown object from a different thread than the one that created it without proper synchronization. It is fundamentally unsafe for concurrent access.

## Data Pipeline

The class serves as a data contract for a specific point in the network data flow.

> **Inbound Flow:**
> Raw TCP Bytes -> Netty ByteBuf -> Hytale Packet Decoder -> **InteractionCooldown.deserialize** -> Game Logic Handler

> **Outbound Flow:**
> Game Logic Event -> `new InteractionCooldown()` -> Hytale Packet Encoder -> **InteractionCooldown.serialize** -> Netty ByteBuf -> Raw TCP Bytes

