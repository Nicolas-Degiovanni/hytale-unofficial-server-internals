---
description: Architectural reference for SyncInteractionChain
---

# SyncInteractionChain

**Package:** com.hypixel.hytale.protocol.packets.interaction
**Type:** Transient

## Definition
```java
// Signature
public class SyncInteractionChain {
```

## Architecture & Concepts
The SyncInteractionChain is a fundamental Data Transfer Object (DTO) within Hytale's interaction protocol. It is not a service or manager, but rather a message that encapsulates the complete state of a player-initiated action or a sequence of related actions. Its primary role is to synchronize complex player interactions between the client and the server, ensuring game state consistency.

The design of this class is heavily optimized for network performance, which dictates its structure:

*   **Fixed and Variable Data Blocks:** The binary layout is split into a fixed-size block for predictable, always-present data (like IDs and flags) and a variable-data block for optional or variable-length fields (like strings or nested objects). This minimizes packet size by avoiding unnecessary padding and markers.
*   **Null Bitfield:** A single byte at the start of the payload, `nullBits`, acts as a bitmask to indicate the presence or absence of nullable fields. This is significantly more efficient than transmitting null terminators or presence flags for each optional field.
*   **Recursive Structure:** The class can contain an array of itself (`newForks`). This allows the protocol to represent complex, branching interactions—where a single action can initiate multiple subsequent, independent action chains—within a single, hierarchical packet structure. This is critical for implementing advanced abilities, combos, or multi-stage interactions.

This object represents a node in a potential tree of interactions. The `chainId` identifies a specific sequence, while `forkedId` and `newForks` allow these sequences to branch, creating sophisticated and predictable gameplay mechanics over a network.

## Lifecycle & Ownership
A SyncInteractionChain object is ephemeral and exists only to be transmitted over the network. Its lifecycle is tightly bound to a single network transaction.

*   **Creation:** An instance is created on the sending side (client or server) to represent a game event. It is populated with the current state of the interaction. On the receiving end, an instance is created by the static `deserialize` method, which reads from a network buffer.
*   **Scope:** The object's scope is transient. It lives only long enough to be serialized into a buffer, or from the moment it is deserialized until it has been fully processed by the game logic. It does not persist between game ticks or network packets.
*   **Destruction:** The object is managed by the Java garbage collector. Once the network handler or game logic system has finished processing the data, all references to the instance are dropped, making it eligible for garbage collection. There are no manual cleanup or `close` methods.

**WARNING:** Due to its transient nature, instances of this class should never be cached or reused for subsequent network messages. Always create a new instance for each distinct interaction chain to be synchronized.

## Internal State & Concurrency
*   **State:** The state of a SyncInteractionChain is fully mutable. All of its fields are public, intended for direct access during the construction (pre-serialization) or processing (post-deserialization) phases. This design prioritizes performance by avoiding getter/setter overhead for what is essentially a structured data container.

*   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, modified, and read by a single thread at a time, typically a Netty network thread or a main game loop thread.

    **CRITICAL WARNING:** Concurrent modification or reading of a SyncInteractionChain instance from multiple threads without explicit, external synchronization will lead to race conditions, data corruption, and unpredictable behavior. The serialization and deserialization methods operate on a Netty ByteBuf and are not internally synchronized.

## API Surface
The primary API consists of the public data fields and the static methods for serialization and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static SyncInteractionChain | O(N) | Constructs a new SyncInteractionChain instance by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the current state of the object into the provided ByteBuf according to the defined binary protocol. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will consume when serialized. Useful for buffer pre-allocation. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only check of the buffer to ensure the data represents a valid object structure without fully deserializing it. Critical for security and stability when processing untrusted packets. |
| clone() | SyncInteractionChain | O(N) | Performs a deep copy of the object and all its nested data structures. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol layer. A handler receives a buffer, validates it, deserializes the object, and passes it to the game logic for processing.

```java
// Example: In a network packet handler
ValidationResult result = SyncInteractionChain.validateStructure(buffer, offset);
if (!result.isValid()) {
    // Disconnect client or log error
    throw new ProtocolException("Invalid SyncInteractionChain packet: " + result.error());
}

SyncInteractionChain interaction = SyncInteractionChain.deserialize(buffer, offset);
gameLogic.processPlayerInteraction(player, interaction);
```

### Anti-Patterns (Do NOT do this)
*   **Stateful Reuse:** Do not hold onto an instance and modify it for a new message. This is error-prone and breaks the assumption of a stateless DTO.
    ```java
    // BAD: Reusing the same object
    SyncInteractionChain interaction = new SyncInteractionChain();
    interaction.chainId = 1;
    network.send(interaction);
    
    interaction.chainId = 2; // This is dangerous and can cause race conditions
    network.send(interaction);
    ```
*   **Concurrent Access:** Do not share an instance between threads without external locking.
    ```java
    // BAD: Shared access without synchronization
    SyncInteractionChain sharedInteraction = ...;
    
    // Thread 1
    executor.submit(() -> sharedInteraction.activeHotbarSlot = 5);
    
    // Thread 2
    executor.submit(() -> network.send(sharedInteraction)); // May send corrupted data
    ```

## Data Pipeline
SyncInteractionChain is a data payload, not a processing stage. It is the information that flows through the pipeline.

**Outbound Flow (e.g., Client to Server)**
> Player Input -> InteractionService creates **SyncInteractionChain** -> `serialize` -> Netty ByteBuf -> Network Stack

**Inbound Flow (e.g., Server to Client)**
> Network Stack -> Netty ByteBuf -> Protocol Decoder calls `deserialize` -> **SyncInteractionChain** instance -> Game State Updater -> World Render

