---
description: Architectural reference for SpawnDeployableFromRaycastInteraction
---

# SpawnDeployableFromRaycastInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class SpawnDeployableFromRaycastInteraction extends SimpleInteraction {
```

## Architecture & Concepts

The SpawnDeployableFromRaycastInteraction class is a concrete implementation of a network protocol message. It is not a service or manager, but rather a passive data structure that represents a specific, stateful player action: spawning a deployable entity at a location determined by a client-side raycast.

This class serves as the canonical data contract between the client and server for this interaction type. Its primary architectural role is to define the precise binary layout for serialization and deserialization over the network. The structure is heavily optimized for performance and low bandwidth, employing a custom binary format with several key characteristics:

*   **Fixed and Variable Data Blocks:** The serialized data is split into a fixed-size header (51 bytes) and a variable-size data block. The header contains primitive types and, crucially, offsets pointing to the location of variable-sized data (like maps or nested objects) within the subsequent block. This allows for efficient, non-sequential parsing.
*   **Nullability Bitfield:** The first byte of the serialized message is a bitfield (`nullBits`). Each bit corresponds to a nullable, complex field. If a bit is 0, the corresponding field is absent from the payload, saving significant space and eliminating the need for null terminators or presence flags for each field.
*   **Defensive Deserialization:** The class includes static methods for validation (`validateStructure`) and size calculation (`computeBytesConsumed`) that operate directly on the byte buffer. This enables the network layer to reject malformed or malicious packets early in the pipeline without the overhead of full object instantiation, acting as a critical security and stability feature.

As a subclass of SimpleInteraction, it integrates into the game's broader Interaction system, which likely functions as a finite state machine. The `next` and `failed` integer fields represent transitions to other interaction states, allowing for complex, chained player actions.

## Lifecycle & Ownership

-   **Creation:** Instances are created through one of two primary pathways:
    1.  **Deserialization:** The most common path. The static `deserialize` method is called by the network protocol layer when an incoming `ByteBuf` is identified as this interaction type.
    2.  **Configuration Loading:** Instantiated by a higher-level system when parsing game asset definitions (e.g., from JSON or HOCON files) that define game interactions.
-   **Scope:** Transient and short-lived. An instance typically exists only for the duration of a single network packet processing cycle or a single game tick. It is created, passed to a handler or system, and then becomes eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or manual cleanup procedures associated with this object.

## Internal State & Concurrency

-   **State:** Highly mutable. All public fields are directly accessible and can be modified after construction. The object is designed as a simple data container to be populated (either by deserialization or a factory) and then read by a consumer. It performs no internal caching.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty event loop or the main game logic thread. Concurrent reads and writes from multiple threads without external synchronization mechanisms will result in undefined behavior, data corruption, and race conditions.

## API Surface

The primary contract of this class is its static serialization interface, not its instance members.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static SpawnDeployableFromRaycastInteraction | O(N) | Constructs an object by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state to a ByteBuf and returns the number of bytes written. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a structural integrity check on the data in a ByteBuf without full deserialization. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total size of a serialized object within a ByteBuf. Used to skip over packets. |
| clone() | SpawnDeployableFromRaycastInteraction | O(N) | Performs a deep copy of the object and its contents. |

## Integration Patterns

### Standard Usage

This object should be treated as an immutable message once created. The network layer deserializes it, and the game logic layer consumes it.

```java
// Executed by a protocol handler on a Netty I/O thread
void handlePacket(ByteBuf packetBuffer) {
    // Validate packet before full deserialization to prevent resource exhaustion
    ValidationResult result = SpawnDeployableFromRaycastInteraction.validateStructure(packetBuffer, 0);
    if (!result.isValid()) {
        // Disconnect client or log error
        return;
    }

    // Full deserialization
    SpawnDeployableFromRaycastInteraction interaction = SpawnDeployableFromRaycastInteraction.deserialize(packetBuffer, 0);

    // Dispatch to the main game thread for processing
    gameLogic.submit(() -> interactionSystem.process(interaction));
}
```

### Anti-Patterns (Do NOT do this)

-   **State Mutation After Deserialization:** Do not modify the state of an instance after it has been deserialized and passed to a consumer system. It should be treated as a read-only snapshot of a remote state or a game data definition. Mutating it can lead to inconsistent behavior.
-   **Manual Instantiation:** Avoid using `new SpawnDeployableFromRaycastInteraction()`. The complex, nullable fields and interdependencies make manual construction highly error-prone. Objects should be created via the `deserialize` method or a dedicated factory that loads from game configuration.
-   **Shared Mutable Access:** Never share an instance of this class between threads without deep cloning it first. Its mutable nature makes it inherently unsafe for concurrent access.

## Data Pipeline

The flow of data through this component is bidirectional, depending on whether the packet is being received or sent.

> **Inbound Flow (Receiving):**
> Raw TCP Packet -> Netty ByteBuf -> **SpawnDeployableFromRaycastInteraction.deserialize** -> Interaction Instance -> Game Interaction System -> World State Update

> **Outbound Flow (Sending):**
> Player Input -> Game Interaction System -> Interaction Instance -> **SpawnDeployableFromRaycastInteraction.serialize** -> Netty ByteBuf -> Raw TCP Packet

