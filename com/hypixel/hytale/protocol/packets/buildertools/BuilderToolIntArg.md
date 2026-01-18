---
description: Architectural reference for BuilderToolIntArg
---

# BuilderToolIntArg

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class BuilderToolIntArg {
```

## Architecture & Concepts
The BuilderToolIntArg class is a specialized Data Transfer Object (DTO) designed for the Hytale network protocol. It does not represent a game entity but rather the metadata for a single integer-based argument within a larger "Builder Tool" definition packet. Its primary role is to define the constraints—default value, minimum, and maximum—for a configurable integer parameter, likely used in an in-game editor or creative mode UI.

Architecturally, this class is a concrete implementation of a fixed-layout data structure. The public constants, such as FIXED_BLOCK_SIZE and MAX_SIZE, explicitly declare its size as 12 bytes (three 4-byte integers). This fixed-size nature is a critical design choice for the protocol layer, as it allows for highly efficient, zero-overhead parsing and buffer allocation without needing to read length prefixes. It is intended to be embedded within larger, more complex packet structures.

### Lifecycle & Ownership
- **Creation:** An instance is created under two primary circumstances:
    1.  **Inbound (Deserialization):** The static factory method `deserialize` is invoked by a higher-level packet decoder when processing an incoming network buffer. The decoder is responsible for managing the buffer's read offset.
    2.  **Outbound (Serialization):** Game logic on the sending side (typically the server) instantiates it directly via its constructor (`new BuilderToolIntArg(...)`) to populate a packet that will be sent to a client.
- **Scope:** The object's lifetime is ephemeral. It is scoped to the lifecycle of its containing network packet. Once the packet has been processed by the game logic (e.g., a UI has been configured with its values), the BuilderToolIntArg instance is no longer referenced.
- **Destruction:** The object is managed by the Java Garbage Collector and is eligible for collection as soon as all references to it and its parent packet are released.

## Internal State & Concurrency
- **State:** The state is fully mutable and exposed through public fields: `defaultValue`, `min`, and `max`. This design prioritizes performance and low allocation overhead over encapsulation, which is a common and acceptable trade-off for internal, short-lived DTOs in a high-performance network layer.
- **Thread Safety:** This class is **not thread-safe**. It is a plain data holder with no internal synchronization mechanisms. It is designed to be created, populated, and read within a single thread, such as a Netty I/O worker thread or the main game logic thread.

**WARNING:** Sharing a BuilderToolIntArg instance across multiple threads without external locking will lead to race conditions and unpredictable behavior. Do not store instances of this class in shared collections or long-lived caches.

## API Surface
The public API is minimal, focusing entirely on serialization, deserialization, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | BuilderToolIntArg | O(1) | **Static Factory.** Constructs an instance by reading 12 bytes from the given buffer at a specific offset. |
| serialize(ByteBuf) | void | O(1) | Writes the three integer fields (12 bytes) into the provided buffer in little-endian format. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **Precondition Check.** Verifies if the buffer contains at least 12 readable bytes from the given offset. |
| computeSize() | int | O(1) | Returns the constant size of the structure, which is always 12. |

## Integration Patterns

### Standard Usage
The class is almost exclusively used by the network protocol layer. A developer would typically interact with it after a parent packet has been fully decoded.

```java
// Hypothetical packet processing logic
// Assume parentPacket has a method to get this argument definition

BuilderToolIntArg rotationArg = parentPacket.getRotationArgument();

// The game UI can now use these constraints to build a slider or input field
int defaultValue = rotationArg.defaultValue;
int minValue = rotationArg.min;
int maxValue = rotationArg.max;
```

### Anti-Patterns (Do NOT do this)
- **Modifying After Deserialization:** Do not modify the fields of an instance received from the network. It represents a static definition from the sender and should be treated as immutable post-creation.
- **Ignoring Validation:** Bypassing `validateStructure` before calling `deserialize` is dangerous. It can lead to an IndexOutOfBoundsException if the network buffer is malformed or truncated, potentially crashing the network thread.

```java
// BAD: Potential for a crash if buffer is too small
BuilderToolIntArg arg = BuilderToolIntArg.deserialize(malformedBuffer, 0);

// GOOD: Always validate first
ValidationResult result = BuilderToolIntArg.validateStructure(buffer, offset);
if (result.isOk()) {
    BuilderToolIntArg arg = BuilderToolIntArg.deserialize(buffer, offset);
}
```

## Data Pipeline
The class serves as a data marshalling and unmarshalling component in the network pipeline.

> **Inbound Flow (Client Receives):**
> Raw TCP Byte Stream -> Netty ByteBuf -> Packet Decoder -> **BuilderToolIntArg.deserialize** -> Game Logic (e.g., UI System)

> **Outbound Flow (Server Sends):**
> Game Logic (e.g., Tool Definition) -> `new BuilderToolIntArg()` -> Packet Encoder -> **BuilderToolIntArg.serialize** -> Netty ByteBuf -> Raw TCP Byte Stream

