---
description: Architectural reference for InteractionChainData
---

# InteractionChainData

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class InteractionChainData {
```

## Architecture & Concepts

InteractionChainData is a specialized Data Transfer Object designed for high-performance network communication. It encapsulates the state of a single, discrete player or entity interaction within the game world, such as clicking on a block, attacking an entity, or using an item.

This class is not a service or manager; it is a pure data container. Its primary role is to act as a payload within a larger network packet, providing a structured format for interaction details that can be efficiently serialized, sent over the network, and deserialized.

The binary serialization format is custom-tailored for efficiency:
*   **Nullable Bit Field:** The first byte of the serialized data is a bitmask (`nullBits`). Each bit corresponds to a nullable field (e.g., hitLocation, hitDetail). This avoids wasting bytes to represent null values for complex types.
*   **Fixed-Size Block:** The initial 61 bytes of the object contain all fixed-size fields and placeholders for nullable complex types. This allows for predictable, high-speed reads of the core data.
*   **Variable-Size Block:** Any variable-length data, such as the hitDetail string, is appended after the fixed-size block. Its presence is dictated by the nullable bit field. This hybrid structure provides both the speed of fixed-offset reads and the flexibility of variable-length data.

This object is a fundamental building block of the client-to-server gameplay protocol, translating player input into a network-transmissible format.

## Lifecycle & Ownership

-   **Creation:** An InteractionChainData object is instantiated under two primary circumstances:
    1.  **Sending Peer (Client):** Created by the input handling system when a player performs an action. Its fields are populated with world-state information at the moment of interaction.
    2.  **Receiving Peer (Server):** Rehydrated from a network buffer by calling the static `deserialize` method within the server's network packet processing pipeline.

-   **Scope:** The object's lifetime is exceptionally brief and tied to the scope of a single event or network packet. It is a transient object, intended to be created, processed, and then immediately discarded.

-   **Destruction:** The object is managed by the Java Garbage Collector. As it is short-lived and holds no external resources, it is typically reclaimed very quickly after the event handler or network processor that consumed it has finished execution.

## Internal State & Concurrency

-   **State:** The internal state is fully mutable. All data fields are public, allowing for direct, low-overhead modification. This design choice prioritizes performance within a single-threaded context, avoiding the overhead of getter and setter method calls.

-   **Thread Safety:** **This class is not thread-safe.** It is designed exclusively for use within a single thread, such as a Netty I/O thread or the main game logic thread. Passing an instance between threads without deep cloning and proper synchronization will result in undefined behavior, data corruption, and race conditions. The public mutable fields make it particularly vulnerable to concurrent modification issues.

## API Surface

The public contract is dominated by static utility methods for serialization and validation, reflecting its role as a network data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | InteractionChainData | O(N) | **[Primary Entry Point]** Static factory. Reconstructs an object from a binary representation in a ByteBuf. N is the length of the variable string data. |
| serialize(buf) | void | O(N) | Encodes the object's state into a ByteBuf according to the custom binary protocol. N is the length of the variable string data. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a lightweight check on a buffer to ensure it contains a structurally valid object without the cost of full deserialization. |
| computeSize() | int | O(N) | Calculates the total number of bytes this object will occupy when serialized. |
| clone() | InteractionChainData | O(1) | Creates a shallow-to-medium copy of the object. Note that the hitDetail string is not deep-copied. |

## Integration Patterns

### Standard Usage

The canonical use case involves deserializing the object from a network buffer within a packet handler, passing the resulting data to the game logic system, and then discarding it.

```java
// Example from a server-side network packet handler
public void handleInteractionPacket(ByteBuf packetBuffer) {
    // Validate the structure before attempting a full deserialization
    ValidationResult result = InteractionChainData.validateStructure(packetBuffer, 0);
    if (!result.isOk()) {
        throw new ProtocolException("Invalid InteractionChainData: " + result.getMessage());
    }

    // Deserialize and process
    InteractionChainData interaction = InteractionChainData.deserialize(packetBuffer, 0);
    gameWorld.processPlayerInteraction(player, interaction);
}
```

### Anti-Patterns (Do NOT do this)

-   **Object Reuse:** Do not reuse an InteractionChainData instance across multiple packets or events. Its mutable state makes it unsafe for pooling or reuse without a full reset, which is error-prone. Always create a new instance for each new interaction.
-   **Cross-Thread Sharing:** Never share an instance of this object between threads. If data must be passed to another thread, create a deep copy using the `clone` method or construct a new, immutable representation of the data.
-   **Manual Serialization:** Do not attempt to read or write the fields to a buffer manually. The serialization logic, especially the handling of the `nullBits` bitmask, is complex. Always use the provided `serialize` and `deserialize` methods to ensure protocol compliance.

## Data Pipeline

InteractionChainData serves as a critical link in the chain that converts player actions into server-side game state changes.

**Client-Side (Serialization Flow):**
> Player Input Event -> Game Logic creates **InteractionChainData** -> `serialize()` -> Netty ByteBuf -> Network Layer

**Server-Side (Deserialization Flow):**
> Network Layer -> Netty ByteBuf -> `deserialize()` -> **InteractionChainData** rehydrated -> Game Logic Processing

