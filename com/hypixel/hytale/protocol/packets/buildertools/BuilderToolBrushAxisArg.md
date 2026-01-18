---
description: Architectural reference for BuilderToolBrushAxisArg
---

# BuilderToolBrushAxisArg

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient

## Definition
```java
// Signature
public class BuilderToolBrushAxisArg {
```

## Architecture & Concepts
The BuilderToolBrushAxisArg class is a specialized Data Transfer Object (DTO) designed for network serialization within the Hytale protocol. It is not a service or a manager, but rather a fundamental data structure representing a single, fixed-size argument for a builder tool command.

Its primary role is to encapsulate the axis constraint for a building brush (e.g., X, Y, Z, or None) into a predictable, 1-byte network representation. The class is a component of a larger, highly structured protocol system, evidenced by its static serialization and validation methods. These methods conform to a consistent pattern used by higher-level packet processors to read and write game data to a Netty ByteBuf.

In essence, this class acts as a type-safe, protocol-aware wrapper for the BrushAxis enum, ensuring that its state can be reliably transmitted between the client and server.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary scenarios:
    1.  **Deserialization:** The static `deserialize` factory method is invoked by a parent packet's deserialization logic when reading an incoming network buffer.
    2.  **Game Logic:** The client or server logic instantiates this class to construct an outgoing command or packet, typically via its constructor.
- **Scope:** This object is extremely short-lived and has a narrow scope. It is designed to exist only for the duration of packet construction or processing. It is a classic transient object.
- **Destruction:** Instances are eligible for garbage collection as soon as the containing packet or command has been processed and is no longer referenced. There is no manual cleanup mechanism.

## Internal State & Concurrency
- **State:** The class holds a single mutable field, `defaultValue` of type BrushAxis. While technically mutable due to its public visibility, it is intended to be treated as immutable after its initial assignment during construction or deserialization. The entire state of the object is contained within this single enum reference.

- **Thread Safety:** This class is **not thread-safe**. The public, non-final `defaultValue` field can be modified without synchronization. This design is acceptable under the assumption that instances are confined to a single thread, such as a Netty I/O worker thread or the main game logic thread.

    **Warning:** Sharing an instance of BuilderToolBrushAxisArg across multiple threads will lead to race conditions and unpredictable behavior. Do not store instances in shared collections or pass them between threads without explicit synchronization, which is considered an anti-pattern for this class.

## API Surface
The public API is designed for integration with an automated packet processing system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BuilderToolBrushAxisArg() | constructor | O(1) | Creates an instance with BrushAxis.None. |
| BuilderToolBrushAxisArg(BrushAxis) | constructor | O(1) | Creates an instance with the specified axis. |
| deserialize(ByteBuf, int) | static BuilderToolBrushAxisArg | O(1) | Factory method to construct an instance from a network buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the internal BrushAxis value as a single byte to the provided buffer. |
| computeSize() | int | O(1) | Returns the fixed size of this object on the wire, which is always 1 byte. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if the buffer has sufficient bytes to contain this object. |
| clone() | BuilderToolBrushAxisArg | O(1) | Creates a shallow copy of the object. |

## Integration Patterns

### Standard Usage
This class is not intended to be used in isolation. It is always a component of a larger packet or command structure. A protocol handler uses its static methods for I/O, while game logic uses its constructors to build commands.

```java
// Example: Constructing a command to send to the server
BuilderToolBrushAxisArg axisArg = new BuilderToolBrushAxisArg(BrushAxis.X);

// The 'axisArg' would then be set on a parent packet object,
// which would later call axisArg.serialize() as part of its own
// serialization routine.
someBuilderToolPacket.setAxisArgument(axisArg);

// ... later, in the network layer ...
someBuilderToolPacket.serialize(outgoingByteBuf);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not modify the `defaultValue` field after the object has been passed to a parent packet or a serialization routine. This can cause a mismatch between the intended state and the serialized data.
- **Cross-Thread Sharing:** Never share an instance between threads. Its mutable, non-synchronized state makes it inherently unsafe for concurrent access.
- **Independent Serialization:** Do not call `serialize` directly. This method is designed to be called by a parent object that manages the ByteBuf write index. Calling it out of context can corrupt the network buffer.

## Data Pipeline
The BuilderToolBrushAxisArg serves as a data container in two opposing pipelines: outbound serialization and inbound deserialization.

**Outbound (e.g., Client to Server)**
> Game Logic Instantiation -> **BuilderToolBrushAxisArg** -> Parent Packet Composition -> ParentPacket.serialize() -> Netty ByteBuf -> Network Socket

**Inbound (e.g., Server to Client)**
> Network Socket -> Netty ByteBuf -> Packet Deserializer -> **BuilderToolBrushAxisArg.deserialize()** -> Parent Packet Hydration -> Game Logic Processing

