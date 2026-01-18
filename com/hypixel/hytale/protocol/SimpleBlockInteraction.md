---
description: Architectural reference for SimpleBlockInteraction
---

# SimpleBlockInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class SimpleBlockInteraction extends SimpleInteraction {
```

## Architecture & Concepts
SimpleBlockInteraction is a Data Transfer Object (DTO) that serves as the in-memory representation of a network packet defining a player's interaction with a block. It is a fundamental component of the Hytale network protocol, designed for high-performance serialization and deserialization.

The core architectural pattern is a custom binary format that separates data into a fixed-size header and a variable-size data block. This design provides two key benefits:
1.  **Predictable Access:** The first 40 bytes contain fixed-size fields, allowing for direct, constant-time reads without parsing.
2.  **Payload Efficiency:** Complex, nullable objects like InteractionEffects or InteractionRules are not serialized directly. Instead, a single byte at the start of the packet, the *nullBits* field, acts as a bitmask indicating their presence. If a field is present, the fixed-size block contains an integer offset pointing to its location in the variable-size data block that follows. This avoids transmitting and parsing unused optional data.

This class is not a service; it is pure data. It encapsulates the complete definition of a block interaction, including visual effects, conditional rules, camera behavior, and state transitions for success or failure.

### Lifecycle & Ownership
-   **Creation:** An instance is created under two primary circumstances:
    1.  **Inbound:** The network layer's packet decoder instantiates the object by calling the static **deserialize** method when a corresponding packet is received from a client or server.
    2.  **Outbound:** Game logic or content authoring systems may instantiate the object via its constructor to define a new interaction that will later be serialized and sent over the network.
-   **Scope:** The object is short-lived and transaction-scoped. It exists only for the duration required to process a single network event.
-   **Destruction:** It holds no external resources and is managed by the Java garbage collector. It becomes eligible for collection as soon as the game logic handler that received it completes its execution.

## Internal State & Concurrency
-   **State:** The object is mutable. Its public fields represent the state of a single, discrete interaction. It does not cache or manage any external state. The state is self-contained.

-   **Thread Safety:** **This class is not thread-safe.** It is a simple data container with no internal locking mechanisms. It is designed to be created, processed, and discarded within a single thread, typically a Netty event loop thread.

    **WARNING:** Sharing an instance of SimpleBlockInteraction across multiple threads without external synchronization will lead to race conditions and unpredictable behavior. Do not store instances in shared collections or pass them between thread contexts without deep cloning.

## API Surface
The primary contract of this class is its binary layout, managed by the serialization and deserialization methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | SimpleBlockInteraction | O(N) | **[Static]** Constructs an object by reading from a ByteBuf. Complexity is proportional to the size of the variable data. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into a ByteBuf using the custom binary format. Returns the number of bytes written. |
| computeSize() | int | O(1) | Calculates the expected byte size of the object if it were to be serialized. Does not perform actual serialization. |
| validateStructure(buf, offset) | ValidationResult | O(N) | **[Static]** Performs a read-only check of a buffer to ensure it contains a structurally valid SimpleBlockInteraction. Does not fully deserialize. |
| clone() | SimpleBlockInteraction | O(N) | Creates a deep copy of the object and its contained data structures. |

## Integration Patterns

### Standard Usage
The class is almost exclusively used by the network protocol decoders and game logic handlers. A handler receives the deserialized object, inspects its properties, and enacts changes in the game world.

```java
// Executed within a network channel handler
void handlePacket(ByteBuf packetData) {
    // The decoder creates the object from the raw buffer
    SimpleBlockInteraction interaction = SimpleBlockInteraction.deserialize(packetData, 0);

    // A game system consumes the object's data
    InteractionSystem interactionSystem = getInteractionSystem();
    interactionSystem.processBlockInteraction(player, interaction);

    // The object is now out of scope and will be garbage collected
}
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Reuse:** Do not reuse an instance of this class for multiple different interactions. Its fields are mutable, and failing to reset all properties can lead to data from a previous interaction leaking into a new one. Create a new instance or use clone for each distinct event.
-   **Cross-Thread Sharing:** Never pass an instance from the network thread to a game logic thread without a synchronization boundary or deep cloning. Modifying the object while another thread is reading it (or serializing it) is a critical race condition.
-   **Ignoring Validation:** Do not deserialize data from an untrusted source without first calling validateStructure. A malicious client could craft a packet with invalid offsets or lengths, causing ProtocolException or, in worse cases, excessive memory allocation.

## Data Pipeline
SimpleBlockInteraction acts as a structured data container in the flow from raw network bytes to game logic.

> Flow:
> Raw TCP Stream -> Netty ByteBuf -> Protocol Packet Decoder -> **SimpleBlockInteraction.deserialize** -> SimpleBlockInteraction Instance -> Game Interaction Handler -> World State Mutation

