---
description: Architectural reference for SetUpdateRate
---

# SetUpdateRate

**Package:** com.hypixel.hytale.protocol.packets.setup
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class SetUpdateRate implements Packet {
```

## Architecture & Concepts
The SetUpdateRate class is a network packet definition that represents a specific command within the Hytale protocol's "setup" phase. Its sole purpose is to communicate the desired server-to-client update frequency, typically measured in ticks per second. This packet is fundamental for synchronizing the client's rendering and simulation loop with the server's state broadcast rate.

As a concrete implementation of the Packet interface, this class provides the core logic for serialization, deserialization, and unique identification required by the network layer. Its design is optimized for high-performance, low-allocation network processing. The static methods, particularly deserialize and validateStructure, allow the Netty-based protocol decoder to operate directly on raw ByteBuf instances without needing to instantiate a SetUpdateRate object prematurely. This stateless processing model is critical for efficiently handling high volumes of network traffic.

The packet has a fixed, minimal size of 4 bytes, reflecting its simple payload: a single 32-bit integer. This simplicity ensures minimal network overhead during the crucial connection setup sequence.

### Lifecycle & Ownership
- **Creation:**
    - **Sending Context (e.g., Server):** Instantiated directly via its constructor, `new SetUpdateRate(rate)`, by a higher-level system responsible for managing client connection parameters.
    - **Receiving Context (e.g., Client):** Instantiated by the network protocol decoder pipeline via the static `deserialize` factory method. The decoder reads the packet ID from the byte stream and dispatches the buffer to this method to construct the object.

- **Scope:** Extremely short-lived. A SetUpdateRate instance is a transient message, not a persistent entity. It exists only for the brief moment it is being serialized into a buffer or from the point of deserialization until it is consumed by a packet handler.

- **Destruction:** The object becomes eligible for garbage collection immediately after its internal data has been read and applied by the receiving system. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The class holds a minimal, mutable state consisting of a single public integer field: updatesPerSecond. The use of a public field is a deliberate design choice for performance-critical DTOs, eliminating the overhead of getter method calls.

- **Thread Safety:** This class is **not thread-safe** and contains no internal synchronization mechanisms. It is designed to be created, serialized, deserialized, and processed within a single thread, typically a Netty I/O worker thread.

    **Warning:** Sharing a SetUpdateRate instance across threads without external synchronization will lead to unpredictable behavior and race conditions.

## API Surface
The public API is designed for interaction with the protocol codec, not for general application logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network ID (29) for this packet type. |
| serialize(ByteBuf buf) | void | O(1) | Writes the updatesPerSecond value into the provided buffer. |
| deserialize(ByteBuf buf, int offset) | SetUpdateRate | O(1) | **Static Factory.** Reads 4 bytes from the buffer and constructs a new SetUpdateRate instance. |
| computeSize() | int | O(1) | Returns the constant size of the packet payload, which is 4 bytes. |
| validateStructure(ByteBuf buffer, int offset) | ValidationResult | O(1) | **Static.** Checks if the buffer contains enough readable bytes for a valid packet. |

## Integration Patterns

### Standard Usage
The class is used exclusively by the network layer. A developer would typically interact with it inside a packet handler.

```java
// Example of a client-side packet handler consuming the packet
public class ClientSetupPacketHandler {
    private final GameLoopScheduler scheduler;

    public void handle(SetUpdateRate packet) {
        // The packet is received, fully formed, from the network pipeline.
        // The handler's responsibility is to apply the data.
        int serverRate = packet.updatesPerSecond;
        this.scheduler.setTargetTickRate(serverRate);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and re-send the same SetUpdateRate instance. These objects are cheap to create and should be treated as immutable after creation. Always instantiate a new packet for each message.
- **Manual Codec Calls:** Do not call `serialize` or `deserialize` directly in application code. These methods are tightly coupled to the protocol's encoder/decoder pipeline, which manages buffer allocation and lifecycle.
- **Cross-Thread Modification:** Do not deserialize a packet on a network thread and pass the mutable instance to a game logic thread for modification. Extract the primitive `updatesPerSecond` value and pass that instead.

## Data Pipeline
The SetUpdateRate packet follows a simple, linear data flow through the network stack.

> **Sending Flow (Server -> Client):**
> Connection Manager -> `new SetUpdateRate(60)` -> Protocol Encoder -> **SetUpdateRate.serialize()** -> Netty Channel -> Raw TCP/UDP Bytes

> **Receiving Flow (Client):**
> Raw TCP/UDP Bytes -> Netty Channel -> Protocol Decoder -> **SetUpdateRate.deserialize()** -> Packet Handler -> Game Loop Scheduler Update

