---
description: Architectural reference for InteractionConfiguration
---

# InteractionConfiguration

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class InteractionConfiguration {
```

## Architecture & Concepts
The InteractionConfiguration class is a data structure that defines the wire format for entity interaction settings within the Hytale network protocol. It is not a service or a manager; it is a Plain Old Java Object (POJO) designed for high-performance serialization and deserialization.

This class serves as a data contract between the client and server. It encapsulates a set of rules—such as interaction distances, priorities, and visual debugging flags—that govern how a player can interact with entities in the game world.

Its most critical architectural feature is the implementation of a custom, non-trivial binary layout. The serialization format is optimized for network efficiency and consists of three main parts:
1.  **Fixed-Size Header:** A 12-byte block containing boolean flags, a null-field bitmask, and offsets to the variable data block.
2.  **Null-Field Bitmask:** The first byte of the header is a bitmask indicating which of the nullable, variable-sized fields (like maps) are present in the payload. This allows the deserializer to skip reading data that was never sent.
3.  **Variable-Size Data Block:** A region following the header where complex data types, such as the `useDistance` and `priorities` maps, are written. The header contains integer offsets pointing to the start of each data structure within this block.

This layout allows for efficient partial reads and validation without needing to parse the entire object structure.

## Lifecycle & Ownership
-   **Creation:** An InteractionConfiguration instance is created in one of two primary scenarios:
    1.  **Deserialization:** The most common path. The static `deserialize` method is called by the network protocol layer to construct an object from an incoming Netty `ByteBuf`.
    2.  **Direct Instantiation:** Game logic may create a new instance using its constructors (`new InteractionConfiguration()`) to define a new set of interaction rules before serializing it for transmission.
-   **Scope:** The object's lifetime is typically short and bound to the scope of a single network packet or a specific game logic operation. It may be held as a component on an entity, in which case its lifetime matches that of the entity.
-   **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The class is **highly mutable**. All of its fields are public, allowing for direct modification. This design prioritizes performance by avoiding getter and setter overhead but sacrifices encapsulation. The state includes simple booleans and nullable `Map` objects for more complex configuration.
-   **Thread Safety:** This class is **not thread-safe**. Its public, mutable fields make it inherently unsafe for concurrent modification.

    **Warning:** Any access or modification of an InteractionConfiguration instance from multiple threads must be externally synchronized. It is designed to be used by a single thread at a time, typically the network thread or the main game loop thread.

## API Surface
The public contract is defined by its public fields and its serialization/validation methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static InteractionConfiguration | O(N) | Constructs an object from a `ByteBuf`. Throws `ProtocolException` on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into a `ByteBuf` using the custom binary format. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the size of a serialized object directly from a buffer without full deserialization. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a structural integrity check on a serialized object in a buffer. Does not create an object. |
| clone() | InteractionConfiguration | O(N) | Performs a deep copy of the object, including its internal maps. |

*N refers to the number of elements in the internal maps.*

## Integration Patterns

### Standard Usage
The primary use case is within the network layer. Game logic constructs or modifies an instance, which is then passed to a network handler for serialization and transmission. On the receiving end, the network handler deserializes the byte stream back into an InteractionConfiguration object.

```java
// Example: Deserializing from a network buffer
// Assume 'incomingBuffer' is a ByteBuf received from the network.
try {
    ValidationResult result = InteractionConfiguration.validateStructure(incomingBuffer, 0);
    if (result.isOk()) {
        InteractionConfiguration config = InteractionConfiguration.deserialize(incomingBuffer, 0);
        // Game logic now uses the 'config' object
        player.applyInteractionConfig(config);
    } else {
        // Handle invalid packet
    }
} catch (ProtocolException e) {
    // Handle deserialization failure (e.g., disconnect client)
}
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Modifying an instance from one thread while another thread is serializing or reading from it will lead to data corruption and unpredictable behavior.
-   **Ignoring Validation:** In a server context, failing to call `validateStructure` on data from an untrusted client before deserialization can expose the server to resource exhaustion attacks (e.g., a client claiming a map has 2 billion entries).
-   **Assuming Non-Null:** The `useDistance` and `priorities` maps are nullable. Always check for null before attempting to access them after deserialization.

## Data Pipeline
InteractionConfiguration acts as a model for data moving between the network and game logic layers.

> **Flow (Server Receiving from Client):**
> Netty `ByteBuf` -> Protocol Decoder -> **InteractionConfiguration.deserialize** -> In-Memory `InteractionConfiguration` Object -> Game Logic Processing -> Entity State Update

> **Flow (Server Sending to Client):**
> Game Logic Creates/Modifies `InteractionConfiguration` -> Protocol Encoder -> **InteractionConfiguration.serialize** -> Netty `ByteBuf` -> TCP/IP Stack

