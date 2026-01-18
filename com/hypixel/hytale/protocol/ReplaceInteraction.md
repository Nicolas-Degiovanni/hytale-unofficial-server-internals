---
description: Architectural reference for ReplaceInteraction
---

# ReplaceInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient / Data Transfer Object

## Definition
```java
// Signature
public class ReplaceInteraction extends Interaction {
```

## Architecture & Concepts
The ReplaceInteraction class is a data transfer object (DTO) that represents a specific type of game interaction within the Hytale network protocol. It is not a service or manager; its sole responsibility is to model the data for a "replace" interaction event for network transmission.

This class, like other protocol objects, employs a highly optimized, custom binary format to minimize network bandwidth. The serialization architecture is divided into two distinct sections within the byte buffer:

1.  **Fixed-Size Block:** A 39-byte header containing primitive fields and offsets. The first byte of this block is a `nullBits` bitmask, which efficiently encodes the presence or absence of nullable, variable-sized fields.
2.  **Variable-Size Block:** A data region following the fixed block. It contains the serialized data for complex objects like maps, arrays, and strings. The fixed block contains integer offsets that point to the start of each corresponding data structure within this variable block.

This design allows for extremely fast deserialization, as the parser can read the fixed block to understand the payload's structure and then jump directly to the required data sections, skipping any fields that are not present.

## Lifecycle & Ownership
- **Creation:** An instance of ReplaceInteraction is created under two circumstances:
    1.  **Outbound:** The game engine instantiates it directly via its constructor when preparing to send an interaction event to a remote peer.
    2.  **Inbound:** The network layer creates an instance by invoking the static `deserialize` factory method when parsing an incoming network packet from a ByteBuf.

- **Scope:** The object's lifetime is ephemeral. It is designed to exist only for the duration of a single network operation. Once serialized to a buffer or deserialized and processed by game logic, it is expected to be dereferenced.

- **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection as soon as it falls out of the scope of the network handler or game logic method that was processing it.

## Internal State & Concurrency
- **State:** The state of a ReplaceInteraction object is fully mutable. Its public fields serve as a container for data that is either about to be serialized or has just been deserialized. It performs no caching and holds no long-term state.

- **Thread Safety:** **WARNING:** This class is not thread-safe. It is designed for thread-confined usage, typically within a single Netty event loop thread or a game tick thread. Concurrent modification from multiple threads will result in data corruption and unpredictable behavior. Any cross-thread usage must be managed with external synchronization.

## API Surface
The public contract is dominated by static methods for serialization, deserialization, and validation, reflecting its role as a network DTO.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf buf) | int | O(N) | Serializes the object's state into the provided ByteBuf. Throws ProtocolException on constraint violations. |
| deserialize(ByteBuf buf, int offset) | static ReplaceInteraction | O(N) | Constructs a new ReplaceInteraction instance by reading from the ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| computeSize() | int | O(N) | Calculates the total byte size the object will occupy when serialized. Useful for pre-allocating buffers. |
| computeBytesConsumed(ByteBuf buf, int offset) | static int | O(N) | Reads a serialized object from a buffer to determine its total size without full deserialization. |
| validateStructure(ByteBuf buffer, int offset) | static ValidationResult | O(N) | Performs a structural validation of the data in the buffer before attempting deserialization. This is a critical security and stability measure. |
| clone() | ReplaceInteraction | O(N) | Creates a deep copy of the object and its contained data structures. |

*N represents the total size of the variable-length fields (strings, maps, arrays).*

## Integration Patterns

### Standard Usage
The primary integration pattern involves a network handler decoding a ByteBuf into a ReplaceInteraction object. Validation should always precede deserialization.

```java
// In a network packet handler
void handlePacket(ByteBuf buffer) {
    int offset = ...; // Determine start of object in buffer

    // 1. Validate before parsing to prevent errors and exploits
    ValidationResult result = ReplaceInteraction.validateStructure(buffer, offset);
    if (!result.isValid()) {
        // Disconnect client or log error
        throw new ProtocolException("Invalid ReplaceInteraction packet: " + result.error());
    }

    // 2. Deserialize into a usable object
    ReplaceInteraction interaction = ReplaceInteraction.deserialize(buffer, offset);

    // 3. Pass to game logic for processing
    gameLogic.processInteraction(interaction);
}
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not deserialize data into a pre-existing ReplaceInteraction instance or reuse a single instance for multiple outbound messages. These objects are lightweight and should be created per-operation to avoid state pollution.
- **Skipping Validation:** Never call `deserialize` on a buffer received from an untrusted source (i.e., any remote client) without first calling `validateStructure`. Doing so exposes the server to malformed packets that can trigger exceptions and cause a denial of service.
- **Cross-Thread Sharing:** Do not pass a ReplaceInteraction instance to another thread after its creation without implementing proper locking. For example, do not deserialize on a network thread and hand the object directly to the main game thread if the network thread might still reference it.

## Data Pipeline
The class acts as a transient container for data moving between the game engine and the network stack.

> **Outbound Flow:**
> Game Logic -> `new ReplaceInteraction(...)` -> **serialize()** -> Netty ByteBuf -> Network Socket

> **Inbound Flow:**
> Network Socket -> Netty ByteBuf -> **validateStructure()** -> **deserialize()** -> ReplaceInteraction Instance -> Game Logic

