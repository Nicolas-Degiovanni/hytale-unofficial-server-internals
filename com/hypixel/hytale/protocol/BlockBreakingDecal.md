---
description: Architectural reference for BlockBreakingDecal
---

# BlockBreakingDecal

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Model

## Definition
```java
// Signature
public class BlockBreakingDecal {
```

## Architecture & Concepts
The BlockBreakingDecal class is a Data Transfer Object (DTO) that operates exclusively within the network protocol layer. It is not a service, manager, or long-lived component. Its sole responsibility is to define the network contract and binary serialization format for the set of texture assets used to render the progressive stages of a block being broken.

This class acts as a low-level data structure, designed for extreme performance and minimal allocation during network I/O. It employs a custom binary format utilizing a bitmask for null fields and VarInts for variable-length data, which is a common pattern in the Hytale protocol to reduce packet size.

It is a leaf component in the protocol object graph, typically embedded within larger, more complex packet objects that manage game state updates. Its existence is entirely subservient to the serialization and deserialization pipeline.

## Lifecycle & Ownership
The lifecycle of a BlockBreakingDecal instance is exceptionally short and tied directly to the processing of a single network packet.

-   **Creation:** An instance is created under two specific conditions:
    1.  **Inbound:** The static factory method *deserialize* is invoked by a higher-level packet decoder when reading data from a Netty ByteBuf. This is the most common creation path.
    2.  **Outbound:** Game logic on the sending side instantiates the object via its constructor, populates the *stageTextures* field, and embeds it within a parent packet for transmission.

-   **Scope:** The object is method-scoped and transient. It exists only for the duration of a single packet's processing cycle. Once its data has been read and transferred to the relevant game systems (like the rendering engine), the object is immediately eligible for garbage collection.

-   **Destruction:** Cleanup is handled exclusively by the Java Garbage Collector. There are no manual resource management or destruction methods. Holding long-term references to these objects is a memory leak anti-pattern.

## Internal State & Concurrency
-   **State:** The internal state is **mutable**. The primary data field, *stageTextures*, is public and can be modified after instantiation. The class is a simple data container with no internal logic to protect its state.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit, external synchronization. Its design assumes it will be created, processed, and discarded within the confines of a single network thread (e.g., a Netty event loop) or a single game-tick thread. Concurrent access will lead to unpredictable behavior and race conditions.

## API Surface
The public API is primarily composed of static utility methods for interacting with raw byte buffers and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static BlockBreakingDecal | O(N) | **Factory Method.** Constructs an object by reading from a ByteBuf. Throws ProtocolException on malformed data or buffer underflow. |
| serialize(buf) | void | O(N) | Serializes the object's state into the provided ByteBuf according to the defined binary protocol. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will consume when serialized. Used for buffer pre-allocation. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the size of a serialized object within a buffer without full deserialization. Used for fast-forwarding a buffer cursor. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a low-cost structural validation on the data in a buffer. Critical for early rejection of malformed packets. |

*N = total bytes of string data in the stageTextures array.*

## Integration Patterns

### Standard Usage
This class should never be interacted with directly by high-level game logic. Its use is confined to the implementation of other packet objects. A parent packet will delegate serialization and deserialization of this data structure.

```java
// Hypothetical usage inside another Packet's serialization logic
public class C08PacketUpdateBlockVisuals extends Packet {
    private BlockBreakingDecal decal;

    @Override
    public void serialize(ByteBuf buf) {
        // The parent packet delegates serialization to the decal instance.
        decal.serialize(buf);
    }

    @Override
    public void deserialize(ByteBuf buf, int offset) {
        // The parent packet uses the static factory to construct the decal.
        this.decal = BlockBreakingDecal.deserialize(buf, offset);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not store instances of BlockBreakingDecal in caches, game state managers, or as member variables of long-lived objects. They are transient DTOs. Extract the string array and discard the container object.
-   **Manual Serialization:** Do not attempt to implement the binary protocol manually. The format is complex (bitmasks, VarInts) and subject to change. Always use the provided *serialize* and *deserialize* methods to guarantee protocol correctness.
-   **Cross-Thread Sharing:** Do not pass an instance from a network thread to a game logic thread. This is a severe concurrency hazard. Instead, pass the deserialized primitive data (the string array).

## Data Pipeline
The BlockBreakingDecal serves as a well-defined data contract in the network pipeline, ensuring both client and server agree on the binary representation of the data.

> **Outbound Flow (Sending):**
> Game State -> `new BlockBreakingDecal(String[])` -> Parent Packet -> `decal.serialize(buf)` -> Netty Encoder -> TCP Socket

> **Inbound Flow (Receiving):**
> TCP Socket -> Netty Decoder -> ByteBuf -> `BlockBreakingDecal.deserialize(buf)` -> Parent Packet -> Game Event Bus -> Renderer State Update

