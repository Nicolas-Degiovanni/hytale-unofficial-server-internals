---
description: Architectural reference for BuilderToolMaskArg
---

# BuilderToolMaskArg

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient

## Definition
```java
// Signature
public class BuilderToolMaskArg {
```

## Architecture & Concepts
The BuilderToolMaskArg class is a low-level Data Transfer Object (DTO) that represents a serializable component within the Hytale network protocol. It is not a service or a manager, but rather a structured container for a single, nullable string value intended for use with in-game builder tools.

Its primary architectural role is to serve as a **Protocol Data Unit (PDU)** fragment. It encapsulates the precise binary layout for its data, including nullability checks and variable-length string encoding. This ensures that clients and servers can communicate builder tool arguments with perfect fidelity. This class exists exclusively within the network layer and should never be exposed to high-level game logic systems. It is typically aggregated within a larger, more complete packet definition.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderToolMaskArg is created under two circumstances:
    1.  **Inbound:** The static `deserialize` method is invoked by a parent packet's deserialization logic when reading a data stream from a Netty ByteBuf.
    2.  **Outbound:** Game logic or a packet factory instantiates it directly using `new BuilderToolMaskArg(value)` to prepare a packet for transmission.

- **Scope:** The object's lifetime is extremely short and is strictly bound to the lifecycle of its parent network packet. It is created, processed, and becomes eligible for garbage collection within the scope of a single network event.

- **Destruction:** The object is managed by the Java Garbage Collector. It is dereferenced and destroyed once the parent packet has been fully processed by the network pipeline and is no longer needed. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** The internal state is **Mutable**. The public `defaultValue` field can be modified after instantiation. However, by convention, an instance should be treated as immutable after being deserialized or before being passed to a serializer.

- **Thread Safety:** This class is **Not Thread-Safe**. It is a simple data holder with no internal locking or synchronization mechanisms. It is designed to be confined to a single thread, typically a Netty I/O worker thread, for the duration of packet serialization or deserialization.

    **WARNING:** Accessing or modifying a BuilderToolMaskArg instance from multiple threads will lead to race conditions and unpredictable network data corruption. Do not share instances across threads.

## API Surface
The public contract is dominated by static methods for interacting with raw byte buffers, reinforcing its role as a serialization helper.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | BuilderToolMaskArg | O(N) | **[Primary Constructor]** Reads from a buffer and constructs a new instance. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided buffer according to the defined binary protocol. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will consume when serialized. Useful for pre-allocating buffers. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a read-only check to verify if the data at the buffer offset represents a valid object, without full deserialization. |

*N represents the length of the defaultValue string.*

## Integration Patterns

### Standard Usage
This class is not meant to be used in isolation. It is composed within a larger packet object that handles a complete network message. The parent packet delegates serialization and deserialization of this specific field to BuilderToolMaskArg.

```java
// Hypothetical Parent Packet (Sending)
BuilderToolMaskPacket packet = new BuilderToolMaskPacket();
packet.setMaskArgument(new BuilderToolMaskArg("hytale:stone"));
networkManager.send(packet); // The packet's serialize method will call arg.serialize()

// Hypothetical Parent Packet (Receiving)
// In the deserializer for BuilderToolMaskPacket:
this.maskArgument = BuilderToolMaskArg.deserialize(buffer, position);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not hold a reference to a BuilderToolMaskArg instance after its parent packet has been processed. Its data is only valid within the context of that single network event.
- **Cross-Thread Sharing:** Do not deserialize a packet on a network thread and pass a reference to its BuilderToolMaskArg to a game logic thread without deep-copying its data into a thread-safe structure.
- **Manual Serialization:** Avoid calling `serialize` directly. This logic should be encapsulated within the `serialize` method of a parent packet to ensure the entire message is constructed correctly.

## Data Pipeline
The flow of data through this component is linear and bidirectional, depending on whether data is being sent or received.

> **Outbound Flow (Client to Server):**
> Game Logic -> `new BuilderToolMaskArg("value")` -> Parent Packet -> Packet.serialize() -> **BuilderToolMaskArg.serialize()** -> Netty ByteBuf -> Network Socket

> **Inbound Flow (Server to Client):**
> Network Socket -> Netty ByteBuf -> Packet Deserializer -> **BuilderToolMaskArg.deserialize()** -> Parent Packet Instance -> Game Logic Event

