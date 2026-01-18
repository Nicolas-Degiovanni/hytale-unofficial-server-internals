---
description: Architectural reference for BuilderToolBrushShapeArg
---

# BuilderToolBrushShapeArg

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class BuilderToolBrushShapeArg {
```

## Architecture & Concepts
The BuilderToolBrushShapeArg class is a low-level data structure within the Hytale network protocol layer. It is not a service or a long-lived component; rather, it serves as a specialized Data Transfer Object (DTO) responsible for the serialization and deserialization of a builder tool's brush shape.

Its primary function is to provide a strongly-typed representation of a single byte on the network wire, mapping it directly to the BrushShape enum. This class encapsulates the precise binary layout for this specific packet argument, abstracting the raw buffer manipulation away from higher-level game logic. It is designed to be a component of a larger, more complex packet structure, never to be used in isolation.

The design emphasizes performance and predictability, with fixed-size calculations and direct buffer access, which is critical for the high-throughput demands of the game's network engine.

### Lifecycle & Ownership
-   **Creation:** Instances are created in two scenarios:
    1.  **Inbound:** The static deserialize method is invoked by a parent packet's deserialization logic when parsing an incoming network buffer.
    2.  **Outbound:** Game logic instantiates it directly when constructing a packet to be sent to the server or client.
-   **Scope:** The object's lifetime is extremely brief and confined to the scope of a single packet processing operation. It is created, its data is extracted or written, and it is then immediately eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or resource release mechanisms required.

## Internal State & Concurrency
-   **State:** The class holds a single mutable public field, defaultValue. While technically mutable, instances should be treated as immutable after their initial creation or deserialization to prevent state corruption during packet processing.
-   **Thread Safety:** This class is **not thread-safe**. Its public field can be modified without synchronization. This is by design, as packet processing is expected to occur within a single, dedicated network thread (e.g., a Netty event loop).

**WARNING:** Do not share instances of BuilderToolBrushShapeArg across threads. Confine usage to the thread responsible for serializing or deserializing a given network packet.

## API Surface
The public API is minimal, focusing exclusively on serialization, deserialization, and size calculation for integration into the protocol pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | BuilderToolBrushShapeArg | O(1) | **[Static]** Reads one byte from the buffer at the given offset and constructs a new instance. |
| serialize(buf) | void | O(1) | Writes the byte value of the internal BrushShape enum to the provided buffer. |
| computeSize() | int | O(1) | Returns the fixed size of this data structure in bytes, which is always 1. |
| validateStructure(buffer, offset) | ValidationResult | O(1) | **[Static]** Verifies that the buffer contains enough readable bytes for a valid structure. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by general game systems. It is exclusively used by the packet serialization and deserialization layer. A parent packet's deserializer would invoke it to parse a segment of the network buffer.

```java
// Example from a hypothetical parent packet's deserialization logic
public static ParentPacket deserialize(ByteBuf buf, int offset) {
    // ... deserialize other fields ...
    int currentOffset = offset + 4; // Assume 4 bytes were read previously

    BuilderToolBrushShapeArg shapeArg = BuilderToolBrushShapeArg.deserialize(buf, currentOffset);
    BrushShape shape = shapeArg.defaultValue;

    // ... use the 'shape' value and continue deserializing ...
    return new ParentPacket(shape);
}
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation After Deserialization:** Modifying the public defaultValue field after an object has been deserialized is a severe anti-pattern. This can lead to a desynchronization between the raw network data and the game state.
-   **Independent Use:** Do not use this class for general application state. Its purpose is strictly for network data transfer. Using it outside the protocol layer couples game logic to the specific binary layout of a network packet.
-   **Assuming Thread Safety:** Never pass an instance to another thread. The lack of synchronization will lead to race conditions and unpredictable behavior.

## Data Pipeline
BuilderToolBrushShapeArg acts as a translation point in the data pipeline, converting raw bytes to a meaningful game enum and vice-versa.

**Inbound Flow (Receiving Data):**
> Flow:
> Netty ByteBuf -> Parent Packet Deserializer -> **BuilderToolBrushShapeArg.deserialize()** -> BrushShape Enum -> Game Logic

**Outbound Flow (Sending Data):**
> Flow:
> Game Logic -> new BuilderToolBrushShapeArg(shape) -> Parent Packet Serializer -> **instance.serialize()** -> Netty ByteBuf

