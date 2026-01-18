---
description: Architectural reference for UpdatePortal
---

# UpdatePortal

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient

## Definition
```java
// Signature
public class UpdatePortal implements Packet {
```

## Architecture & Concepts
The UpdatePortal class is a Data Transfer Object (DTO) that represents a network message within Hytale's protocol layer. Its sole purpose is to communicate changes to in-game portal entities between the server and client. As an implementation of the Packet interface, it adheres to a strict contract for serialization, deserialization, and size computation, making it a fundamental component of the network pipeline.

Architecturally, this packet is designed for high performance and low overhead. It employs several common network protocol optimizations:
- **Bitmasking:** A single byte, referred to as nullBits, is used as a bitmask to efficiently encode the presence or absence of nullable fields (state, definition). This avoids the need for larger, fixed-size presence flags.
- **Hybrid Layout:** The packet structure is a combination of a fixed-size block and a variable-size block. The initial 6 bytes contain the nullBits and the fixed-size PortalState, while the subsequent data for the PortalDef is variable in length. This allows for efficient, direct memory access for the fixed portion while accommodating dynamic data.
- **Static Deserialization:** The deserialization logic is implemented as a static factory method, deserialize. This decouples the process of object creation from the raw network buffer, allowing the network layer to construct packet objects without prior instantiation.

This class is a pure data container and contains no business logic. It acts as a structured, type-safe representation of a specific network command.

## Lifecycle & Ownership
- **Creation:** An UpdatePortal instance is created under two circumstances:
    1. **Outbound:** By game logic on the sending endpoint (e.g., the server) when a portal's state or definition needs to be transmitted.
    2. **Inbound:** By the network protocol layer on the receiving endpoint, specifically when the static deserialize method is invoked on a raw ByteBuf.
- **Scope:** The object's lifetime is exceptionally short and tied to a single network event. It is created, processed, and immediately becomes eligible for garbage collection. It is not intended to be stored in long-term collections or cached.
- **Destruction:** The object is managed by the Java Garbage Collector. There are no native resources or manual cleanup procedures required.

## Internal State & Concurrency
- **State:** The class holds a mutable state consisting of a PortalState and a PortalDef. These fields are public and can be modified after instantiation. The object's primary purpose is to be a transient container for this data during network transport.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data structure with no internal locking or synchronization mechanisms. It is designed to be created, populated, and read within the confines of a single thread, typically a Netty I/O worker thread.

**WARNING:** Sharing an UpdatePortal instance across multiple threads without explicit external synchronization will lead to race conditions and unpredictable behavior. Do not store instances of this packet in shared state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static UpdatePortal | O(N) | Constructs an UpdatePortal object by reading from a ByteBuf at a given offset. N is the size of the variable data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf. N is the size of the variable data. |
| computeSize() | int | O(1) | Calculates the total byte size the packet will occupy when serialized. Does not perform the actual serialization. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a read-only check on a ByteBuf to ensure it contains a structurally valid UpdatePortal packet. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the low-level network handling layer. A packet dispatcher receives a buffer, reads the packet ID, and invokes the corresponding static deserialize method.

```java
// In a network handler receiving a ByteBuf
final int PACKET_ID = 229;

// ... logic to identify packet ID from buffer ...

if (id == PACKET_ID) {
    UpdatePortal packet = UpdatePortal.deserialize(buffer, offset);
    // Pass the fully-formed packet to the game logic thread
    gameLogic.handlePortalUpdate(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not hold references to UpdatePortal objects after they have been processed. They represent a point-in-time event, not a persistent state. Caching them can lead to memory leaks and usage of stale data.
- **Cross-Thread Modification:** Do not create a packet on one thread and serialize it on another without proper memory fencing. The internal state is not volatile.

## Data Pipeline
The flow of data through this component is linear and unidirectional for both sending and receiving.

> **Outbound Flow (Sending):**
> Game Logic -> `new UpdatePortal(state, def)` -> `serialize(ByteBuf)` -> Netty Channel -> Network

> **Inbound Flow (Receiving):**
> Network -> Netty Channel -> `ByteBuf` -> Packet Dispatcher -> **`UpdatePortal.deserialize(ByteBuf)`** -> Network Handler -> Game Logic Update

