---
description: Architectural reference for WieldingInteraction
---

# WieldingInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Protocol Data Structure

## Definition
```java
// Signature
public class WieldingInteraction extends ChargingInteraction {
```

## Architecture & Concepts
The WieldingInteraction class is a data structure that represents a specific type of player action within the Hytale network protocol. It is not a service or manager; it is a passive Data Transfer Object (DTO) designed exclusively for serialization and deserialization of game state between the client and server. This class defines the properties of an interaction that involves wielding an item, such as swinging a sword or using a tool, inheriting base properties from ChargingInteraction.

The core architectural pattern is a highly optimized custom binary serialization format designed to minimize network bandwidth and processing overhead. The structure of a serialized WieldingInteraction payload is non-trivial and follows a specific layout:

1.  **Nullable Bit Field:** A 2-byte bitmask at the beginning of the payload. Each bit corresponds to a nullable complex field (e.g., effects, rules, camera). If the bit is set, the field is present in the data stream; otherwise, it is omitted entirely, saving significant space.
2.  **Fixed-Size Data Block:** A contiguous block of 58 bytes containing primitive fields like floats, booleans, and integers. This fixed layout allows for extremely fast, direct-offset reads without parsing.
3.  **Variable Data Offset Table:** Following the fixed block, there is a table of 8 integer offsets. Each offset points to the start of a corresponding variable-sized data structure (e.g., maps, arrays, nested objects) within the variable data block.
4.  **Variable-Size Data Block:** Appended at the end of the message, this block contains the actual data for the complex, variable-sized fields. This separation prevents a change in the size of one field from affecting the memory offsets of all subsequent fields.

This hybrid fixed-and-variable layout is a deliberate performance choice, trading implementation complexity for significant gains in network efficiency and deserialization speed, which are critical for real-time gameplay.

### Lifecycle & Ownership
- **Creation:** An instance of WieldingInteraction is created in one of two scenarios:
    1.  **Deserialization (Receiving):** The primary creation path. The static factory method `deserialize` is invoked by a network channel handler when a corresponding packet is received. It reads from a Netty ByteBuf and constructs a fully populated object.
    2.  **Instantiation (Sending):** A new instance is created using its constructor on the sending side (e.g., the server). Game logic populates its public fields before passing it to the network layer for serialization via the `serialize` method.

- **Scope:** The lifetime of a WieldingInteraction object is extremely short and transient. It is scoped to the processing of a single network packet. It is created, its data is consumed by the game logic, and it is then immediately eligible for garbage collection.

- **Destruction:** The object is managed by the Java Garbage Collector. There are no manual cleanup or `close` methods. References must not be held beyond the immediate scope of packet processing to prevent memory leaks.

## Internal State & Concurrency
- **State:** The class is a mutable container of public fields. Its purpose is to hold state, not to manage it. The data represents a point-in-time snapshot of a game interaction.

- **Thread Safety:** **This class is not thread-safe.** Instances are designed to be created, populated, and read within a single thread context (typically a Netty I/O thread or the main game logic thread). Concurrent modification from multiple threads will result in race conditions and data corruption. Any handoff between threads, such as from a network thread to the main game loop, must be managed externally using thread-safe queues or other synchronization mechanisms.

## API Surface
The public API is dominated by static methods for serialization and validation, reflecting its role as a data protocol definition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | WieldingInteraction | O(N) | **[Critical]** Static factory. Reconstructs the object from a binary payload in a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | **[Critical]** Writes the object's state into a ByteBuf according to the defined binary format. Returns the number of bytes written. |
| validateStructure(buf, offset) | ValidationResult | O(M) | **[Security]** Performs sanity checks on a buffer before deserialization. Verifies offsets, lengths, and bounds to prevent crashes. |
| computeBytesConsumed(buf, offset) | int | O(M) | Calculates the total size of a serialized object in a buffer without full deserialization. Used for advancing a buffer's read pointer. |
| clone() | WieldingInteraction | O(N) | Creates a deep copy of the object and its contained data structures. |

*N = total size of serialized data. M = number of variable-sized fields.*

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol layer. A packet decoder would first validate the buffer and then deserialize it to be passed to game logic handlers.

```java
// Executed within a network channel handler
// WARNING: Assumes buffer contains a valid WieldingInteraction at offset 0

ValidationResult result = WieldingInteraction.validateStructure(packetBuffer, 0);

if (!result.isValid()) {
    // Disconnect client or log error; do not attempt to deserialize
    throw new ProtocolException("Invalid WieldingInteraction structure: " + result.error());
}

WieldingInteraction interaction = WieldingInteraction.deserialize(packetBuffer, 0);

// Pass the fully formed object to the main game thread for processing
gameLogicQueue.post(interaction);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not maintain references to WieldingInteraction objects in persistent game state. They represent transient network events. If the data is needed long-term, copy its values into dedicated game state components.
- **Ignoring Validation:** Never call `deserialize` on a buffer received from an external source without first calling `validateStructure`. Bypassing validation exposes the server to buffer overflow vulnerabilities and denial-of-service attacks from malformed packets.
- **Concurrent Modification:** Do not share an instance across threads. If data must be passed between threads, either create a deep copy using `clone()` or ensure a safe publication mechanism is used.

## Data Pipeline
WieldingInteraction serves as a data contract for a specific step in the network-to-gameplay pipeline. It has no logic of its own but is essential for translating raw bytes into a structured, usable format for the game engine.

> Flow (Server to Client):
> Server Game State -> **new WieldingInteraction()** -> serialize() -> TCP Packet -> Client Network Decoder -> **WieldingInteraction.deserialize()** -> Client Game Logic Update

