---
description: Architectural reference for RefillContainerInteraction
---

# RefillContainerInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class RefillContainerInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The RefillContainerInteraction class is a Data Transfer Object (DTO) that defines the complete data contract for a specific type of player-world interaction. It exists within the core network protocol layer and represents a serializable message sent between the server and client. Its primary function is to encapsulate all parameters required to execute the logic when a player interacts with a container that can be refilled, such as a well or cauldron.

Architecturally, this class is designed for high-performance network communication. It employs a custom binary serialization format optimized for size and speed, which consists of two main parts:
1.  **Fixed-Size Block:** A 44-byte header containing primitive types and offsets. This allows for immediate, predictable access to core data.
2.  **Variable-Size Block:** A subsequent data region containing complex, nullable objects like lists and nested structures. The offsets in the fixed-size block point to the start of these structures within this region.

A single leading byte, **nullBits**, acts as a bitfield to efficiently declare which of the nullable, variable-sized fields are present in the payload, eliminating the need to transmit empty data. This entire structure is designed to be read from and written to a Netty ByteBuf without intermediate object allocations where possible.

This class is not a service or manager; it is pure state. It is created by a network decoder, interpreted by the game logic, and then discarded.

## Lifecycle & Ownership
-   **Creation:** An instance is almost exclusively created by the static factory method **deserialize** when a network packet containing this interaction data is being decoded by a client or server. On the server, it may be instantiated directly via its constructor before being serialized and sent to a client.
-   **Scope:** The object's lifetime is extremely short and is scoped to the processing of a single network packet. It is created, read by the relevant game systems, and then becomes eligible for garbage collection.
-   **Destruction:** There is no manual destruction. The Java Garbage Collector reclaims the memory once it is no longer referenced, typically after the game tick or network event handler that processed it completes.

## Internal State & Concurrency
-   **State:** The internal state is fully mutable. All fields are public and are populated during the deserialization process. After deserialization, the object should be treated as an immutable data record by consuming systems.

-   **Thread Safety:** **WARNING:** This class is fundamentally **not thread-safe**. It contains no locks, volatile fields, or other concurrency primitives. Its instances are intended to be created, processed, and discarded within the confines of a single thread, such as a Netty I/O worker thread or the main game loop thread. Sharing an instance across threads will result in memory visibility issues and race conditions.

## API Surface
The primary contract of this class is its binary layout, managed by the static serialization and validation methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | RefillContainerInteraction | O(N) | **[Factory]** Constructs an object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into the provided ByteBuf. Returns the number of bytes written. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a security and integrity check on the raw buffer data *before* deserialization to prevent crashes or exploits from malformed packets. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size of a serialized object directly from a ByteBuf without full deserialization. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. |
| clone() | RefillContainerInteraction | O(N) | Creates a deep copy of the object and its contents. |

## Integration Patterns

### Standard Usage
The canonical use case is within a network packet handler or decoder. The system reads the raw bytes, validates the structure, and then deserializes it into an object for the game logic to consume.

```java
// In a network decoder or packet handler
ByteBuf packetPayload = ...;
int offset = ...;

// 1. ALWAYS validate untrusted data first.
ValidationResult result = RefillContainerInteraction.validateStructure(packetPayload, offset);
if (!result.isValid()) {
    throw new ProtocolException("Invalid RefillContainerInteraction: " + result.error());
}

// 2. Deserialize into a usable object.
RefillContainerInteraction interaction = RefillContainerInteraction.deserialize(packetPayload, offset);

// 3. Pass the immutable data to the game logic.
gameLogic.handleInteraction(interaction);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation After Deserialization:** Do not modify the fields of an instance after it has been deserialized. Consuming systems should treat it as an immutable record for the duration of its scope.
-   **Ignoring Validation:** Never call deserialize on a buffer received from an external source without first calling validateStructure. Failure to do so exposes the system to buffer overflows and other parsing vulnerabilities.
-   **Cross-Thread Sharing:** Do not pass an instance of this object to another thread. If state must be shared, extract the primitive values into a thread-safe structure.

## Data Pipeline
The RefillContainerInteraction serves as a data payload that flows from the network hardware up to the game logic.

> Flow:
> Network Socket -> Netty ByteBuf -> **RefillContainerInteraction.validateStructure** -> **RefillContainerInteraction.deserialize** -> Game Logic System -> World State Update

