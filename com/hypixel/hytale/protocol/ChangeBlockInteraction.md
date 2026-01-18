---
description: Architectural reference for ChangeBlockInteraction
---

# ChangeBlockInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Protocol Data Object

## Definition
```java
// Signature
public class ChangeBlockInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The ChangeBlockInteraction class is a specialized data structure that represents a specific type of player-world interaction packet. It is not a service or manager, but rather a Plain Old Java Object (POJO) designed for the efficient serialization and deserialization of data transmitted between the client and server.

Its primary architectural role is to serve as a strongly-typed representation of a network payload that instructs the game engine to modify one or more blocks in the world. It extends SimpleBlockInteraction, inheriting a common set of interaction properties while adding its own specific payload, most notably the `blockChanges` map.

The class is a critical component of the network protocol layer. Its design prioritizes network performance and data integrity through a custom binary format. This format employs a bitmask (`nullBits`) to conditionally include optional fields, and a two-part structure consisting of a fixed-size block for primitive types and a variable-size block for complex objects. This ensures that packets are as compact as possible.

### Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the protocol layer. On the receiving end (e.g., a client), an instance is materialized by the static `deserialize` method, which reads directly from a Netty ByteBuf. On the sending end (e.g., a server), an instance is created programmatically, populated with game state data, and then passed to the `serialize` method.
- **Scope:** Transient and short-lived. The lifetime of a ChangeBlockInteraction object is typically confined to the scope of a single network packet processing cycle. It is created, read by a game logic handler, and then immediately becomes eligible for garbage collection.
- **Destruction:** There are no explicit destruction or cleanup methods. The Java Garbage Collector manages memory once all references to the instance are dropped.

## Internal State & Concurrency
- **State:** The object is fundamentally mutable. Its public fields are intended to be populated once during deserialization or programmatic creation, and then read by a consumer. It contains no internal logic for managing or protecting its state.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, processed, and discarded within a single thread, such as a Netty I/O worker thread or the main game loop thread. Sharing instances of ChangeBlockInteraction across threads without explicit, external synchronization mechanisms is an unsupported pattern and will result in data corruption and undefined behavior.

## API Surface
The public contract is dominated by static utility methods for protocol handling and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ChangeBlockInteraction | O(N) | Constructs an object by reading from a binary buffer. Throws ProtocolException on malformed data or buffer underflow. |
| serialize(buf) | int | O(N) | Writes the object's state into a binary buffer. Returns the total number of bytes written. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Scans a buffer to verify data integrity without the overhead of full object deserialization. Critical for security and protocol stability. |
| computeSize() | int | O(N) | Calculates the exact size in bytes the object will occupy when serialized. Used for pre-allocating buffers. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the size of a serialized object already present in a buffer. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol handlers. A handler receives a raw buffer, identifies the packet type, and uses the static `deserialize` method to create a usable object for the game logic layer.

```java
// Example within a hypothetical packet handler
void handlePacket(ByteBuf packetBuffer) {
    // The offset would be determined by the packet framing protocol
    int payloadOffset = ...;

    // Validate before deserializing to prevent resource exhaustion attacks
    ValidationResult result = ChangeBlockInteraction.validateStructure(packetBuffer, payloadOffset);
    if (!result.isValid()) {
        throw new MalformedPacketException(result.error());
    }

    // Deserialize into a strongly-typed object
    ChangeBlockInteraction interaction = ChangeBlockInteraction.deserialize(packetBuffer, payloadOffset);

    // Pass the data object to the game engine for processing
    gameWorld.processInteraction(interaction);
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Deserialization:** Do not modify the state of a deserialized object. Treat it as an immutable data record once it has been read from the network. Modifying it can lead to inconsistent state if the object is passed to multiple systems.
- **Instance Caching or Re-use:** These objects are lightweight and designed to be transient. Do not cache or re-use instances for subsequent packets. This is a direct path to state corruption. Create a new instance for each discrete network message.
- **Cross-Thread Sharing:** **CRITICAL WARNING:** Never pass an instance of this class from a network thread to a game logic thread without a proper synchronization or queuing mechanism. Direct sharing will cause severe concurrency issues.

## Data Pipeline
ChangeBlockInteraction acts as a data transfer object that carries information from the raw network layer to the application logic.

> **Client-Side Ingress Flow:**
> Raw TCP Stream -> Netty ByteBuf -> Protocol Decoder -> **ChangeBlockInteraction.deserialize** -> **ChangeBlockInteraction Instance** -> Game Event Handler -> World Renderer Update

> **Server-Side Egress Flow:**
> Game Logic (e.g., player action) -> **ChangeBlockInteraction Instance** -> **ChangeBlockInteraction.serialize** -> Protocol Encoder -> Netty ByteBuf -> Raw TCP Stream

