---
description: Architectural reference for ProjectileInteraction
---

# ProjectileInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ProjectileInteraction extends SimpleInteraction {
```

## Architecture & Concepts

The ProjectileInteraction class is a data structure that represents a specific type of network message within the Hytale protocol. It is not a service or manager, but rather a Plain Old Java Object (POJO) designed for high-performance serialization and deserialization.

Architecturally, this class embodies a custom, highly optimized binary protocol format. Its primary role is to act as a Data Transfer Object (DTO) between the client and server for game events related to projectile interactions. The structure is designed to minimize network bandwidth and CPU overhead during parsing. This is achieved through a hybrid layout:

1.  **Fixed-Size Block:** A 19-byte block at the beginning of the serialized data contains core, always-present fields like speeds, flags, and state transition identifiers (next, failed).
2.  **Offset Table:** A 24-byte block that contains integer offsets pointing to the location of variable-sized data within the payload.
3.  **Variable-Size Block:** A region following the fixed block where complex, optional data structures (like effects, rules, and settings) are stored.

A key feature is the `nullBits` byte, a bitmask used to efficiently encode the presence or absence of nullable, variable-sized fields. This avoids the need to transmit empty data for optional components, a critical optimization for game networking.

## Lifecycle & Ownership

-   **Creation:** An instance of ProjectileInteraction is created under two circumstances:
    1.  **Serialization (Sending):** The game logic instantiates it via its constructor when a projectile event occurs that must be communicated over the network.
    2.  **Deserialization (Receiving):** The static `deserialize` method is called by a network pipeline handler (e.g., a Netty ChannelInboundHandler) to construct an object from an incoming ByteBuf.
-   **Scope:** The object's lifetime is intentionally brief. It exists only for the duration of a single network event processing cycle. On the sending side, it is typically eligible for garbage collection immediately after being serialized into a buffer. On the receiving side, it is used by the game logic to update the world state and then discarded.
-   **Destruction:** Cleanup is managed by the Java Garbage Collector. The class holds no native resources and does not require explicit destruction.

## Internal State & Concurrency

-   **State:** The class is **highly mutable**. Its public fields serve as a direct container for the deserialized data. It does not perform any caching or lazy loading; its state is a direct 1-to-1 mapping of the data read from the network buffer.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. It is designed to be created, populated, and read within a single thread, such as a Netty event loop thread or the main game engine thread.

    **WARNING:** Concurrent modification of a ProjectileInteraction instance from multiple threads will lead to data corruption and unpredictable behavior. All access must be externally synchronized if multi-threaded use is unavoidable.

## API Surface

The public contract is dominated by static methods for protocol handling rather than instance methods for business logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | ProjectileInteraction | O(N) | **Primary Entry Point.** Constructs an object by parsing binary data from a ByteBuf. N is the size of the variable data. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into a ByteBuf according to the custom binary protocol. Returns the number of bytes written. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a read-only check of the binary data in a buffer to ensure it conforms to the expected structure without fully deserializing it. Crucial for security and preventing parsing errors. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size of a serialized ProjectileInteraction message within a buffer by reading its headers and variable-length field sizes. |
| computeSize() | int | O(1) | Calculates the expected serialized size of the current in-memory object. Used for buffer pre-allocation. |
| clone() | ProjectileInteraction | O(N) | Performs a deep copy of the object and its contained data structures. |

## Integration Patterns

### Standard Usage

The class is intended to be used by the network layer. A protocol handler receives a buffer, validates it, and deserializes it into a ProjectileInteraction object, which is then passed to the game logic.

```java
// Executed within a Netty channel handler or similar network processor
ValidationResult result = ProjectileInteraction.validateStructure(incomingBuffer, offset);

if (!result.isValid()) {
    // Handle or log the corrupt packet and disconnect the client
    throw new ProtocolException("Invalid ProjectileInteraction: " + result.error());
}

ProjectileInteraction interaction = ProjectileInteraction.deserialize(incomingBuffer, offset);

// Pass the fully-formed object to the game simulation thread for processing
gameLogic.handleInteraction(interaction);
```

### Anti-Patterns (Do NOT do this)

-   **Object Re-use:** Do not attempt to reuse ProjectileInteraction objects across multiple network messages. Their state is specific to a single event, and re-using them can lead to subtle bugs from leftover data. Always create a new instance.
-   **Manual Deserialization:** Do not attempt to read fields manually from the ByteBuf. The binary layout is complex, involving offsets and bitmasks. Always use the static `deserialize` method, which correctly implements the protocol.
-   **Ignoring Validation:** Skipping the `validateStructure` step in a server environment is dangerous. This can expose the server to denial-of-service attacks via malformed packets that trigger exceptions or infinite loops during deserialization.

## Data Pipeline

The ProjectileInteraction class is a critical link in the network data pipeline, translating between in-memory game state and the on-the-wire binary format.

> **Outbound Flow (Serialization):**
> Game Event -> `new ProjectileInteraction(...)` -> **`serialize(ByteBuf)`** -> Netty Channel -> Network Socket

> **Inbound Flow (Deserialization):**
> Network Socket -> Netty Channel -> ByteBuf -> **`deserialize(ByteBuf)`** -> Game Logic -> World State Update

