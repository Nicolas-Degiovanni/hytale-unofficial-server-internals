---
description: Architectural reference for SimpleInteraction
---

# SimpleInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class SimpleInteraction extends Interaction {
```

## Architecture & Concepts
The SimpleInteraction class is a concrete Data Transfer Object (DTO) that represents a specific, stateful player or entity interaction within the Hytale protocol layer. It is not a service or manager; its sole purpose is to encapsulate the data defining an interaction for network transmission between the client and server.

Its design is heavily optimized for network performance and protocol robustness, centered around a custom binary serialization format. The structure of a serialized SimpleInteraction object is divided into two main parts:

1.  **Fixed-Size Block (39 bytes):** This initial block contains primitive fields that are always present, such as identifiers, flags, and floating-point values. Its predictable size allows for extremely fast initial parsing.
2.  **Variable-Size Block:** This block contains complex, nullable, or variable-length data structures like maps, arrays, and nested objects.

To bridge these two blocks, the protocol employs a sophisticated offset-based system. The first byte of the entire structure is a **null-bit field**, where each bit flags the presence or absence of a corresponding variable-sized field. If a field is present, the fixed block contains an integer *offset* that points to the start of that field's data within the variable block. This architecture provides significant advantages:

*   **Efficiency:** Payloads are kept minimal, as space is not allocated for absent optional fields.
*   **Performance:** Parsers can read the fixed block and immediately know the full layout, and can even skip deserializing fields they are not interested in.
*   **Forward Compatibility:** New optional fields can be added to the protocol by assigning a new bit in the null-bit field without breaking older clients.

## Lifecycle & Ownership
- **Creation:** A SimpleInteraction instance is created through one of two primary pathways:
    1.  **Deserialization:** The static factory method *deserialize* is called by the network protocol decoder when an incoming network packet of the corresponding type is identified. This is the most common creation path.
    2.  **Direct Instantiation:** Game logic instantiates the class directly using one of its constructors when preparing an outbound message to be sent to the remote peer.

- **Scope:** The object is ephemeral and has a very short lifecycle. It is scoped to the processing of a single network packet. It is created, read by a handler, and then becomes eligible for garbage collection.

- **Destruction:** The object is managed by the Java Garbage Collector. There are no manual cleanup or close methods. Its lifecycle is complete once the network packet handler finishes its execution and no longer holds a reference to the instance.

## Internal State & Concurrency
- **State:** The SimpleInteraction object is **highly mutable**. Its public fields can be modified after construction. It is designed as a simple data container and does not enforce immutability.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. It is designed to be created, populated, and read within a single thread, typically a Netty I/O worker thread.

    **WARNING:** Sharing a SimpleInteraction instance across multiple threads without external, explicit synchronization will lead to memory consistency errors and race conditions. Treat instances as thread-local.

## API Surface
The primary contract is defined by its static utility methods for serialization and the instance methods for writing data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | SimpleInteraction | O(N) | **Static Factory.** Deserializes an object from the given ByteBuf at a specific offset. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Serializes the object's state into the provided ByteBuf. Returns the number of bytes written. |
| validateStructure(buf, offset) | ValidationResult | O(C) | **Static Utility.** Performs a structural pre-validation of the data in a buffer without full deserialization. Critical for security and preventing parsing errors. |
| computeBytesConsumed(buf, offset) | int | O(C) | **Static Utility.** Calculates the total size of a serialized object within a buffer by reading its headers and offsets. Does not perform a full parse. |
| clone() | SimpleInteraction | O(N) | Creates a deep copy of the object, including its internal collections. |

*N = total size of serialized data in bytes. C = number of variable-sized fields present.*

## Integration Patterns

### Standard Usage
The class is almost exclusively used by the network layer. A handler receives a buffer, validates it, and then deserializes the object to be passed to game logic.

```java
// Executed within a network packet handler
void handleInteractionPacket(ByteBuf packetBuffer) {
    int offset = ...; // Start of the object in the buffer

    // Pre-flight check to ensure data is well-formed
    ValidationResult result = SimpleInteraction.validateStructure(packetBuffer, offset);
    if (!result.isValid()) {
        throw new ProtocolException("Invalid SimpleInteraction: " + result.error());
    }

    // Deserialize into a usable object
    SimpleInteraction interaction = SimpleInteraction.deserialize(packetBuffer, offset);

    // Pass to game logic for processing
    gameWorld.processInteraction(interaction);
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and re-serialize the same SimpleInteraction instance. These objects should be treated as immutable messages once created or deserialized. Create a new instance for each new outbound message.
- **Concurrent Access:** Do not access a SimpleInteraction object from a separate thread while the network thread is deserializing it or after it has been passed to another system. If data must be shared, copy its values into a thread-safe structure.
- **Skipping Validation:** In a server context, never call *deserialize* on untrusted client data without first calling *validateStructure*. Bypassing this check exposes the server to malformed packets that could trigger exceptions or exploits.

## Data Pipeline
The SimpleInteraction class is a key payload in the network data flow.

> **Inbound Flow (e.g., Server receiving from Client):**
> Raw TCP Bytes -> Netty ByteBuf -> Protocol Decoder -> **SimpleInteraction.validateStructure** -> **SimpleInteraction.deserialize** -> Game Logic Handler

> **Outbound Flow (e.g., Server sending to Client):**
> Game Event -> `new SimpleInteraction(...)` -> Protocol Encoder -> **instance.serialize(ByteBuf)** -> Netty ByteBuf -> Raw TCP Bytes

