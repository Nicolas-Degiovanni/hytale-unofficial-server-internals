---
description: Architectural reference for InteractionPriority
---

# InteractionPriority

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object / Transient

## Definition
```java
// Signature
public class InteractionPriority {
```

## Architecture & Concepts
The InteractionPriority class is a specialized Data Transfer Object (DTO) designed for high-performance network communication within the Hytale protocol layer. Its sole responsibility is to encapsulate and transport a map of interaction types, represented by the PrioritySlot enum, to their corresponding integer-based priority levels.

This class is not a component of the core game logic. Instead, it serves as a data container, a fundamental building block for network packets that define interaction rules between entities. Its design prioritizes serialization efficiency and minimal memory overhead over encapsulation.

The binary format is custom and highly optimized for network transit:
- A single-byte bitfield indicates the nullability of the internal map.
- The map size is encoded using a variable-length integer (VarInt) to conserve bandwidth for small maps.
- Data is written and read in Little Endian format, consistent with the engine's protocol specification.

Static methods like deserialize, computeBytesConsumed, and validateStructure operate directly on Netty ByteBuf instances, allowing the network layer to decode or inspect packet data without needing to fully instantiate an object.

## Lifecycle & Ownership
- **Creation:** Instances are primarily created by the protocol decoding pipeline via the static `InteractionPriority.deserialize` method when processing an incoming network buffer. Game systems may also construct instances using the public constructor when preparing an outgoing packet.
- **Scope:** Transient. The lifetime of an InteractionPriority object is typically confined to the processing of a single network packet. It is created, its data is consumed by a handler or game system, and it is then immediately eligible for garbage collection.
- **Destruction:** Managed by the Java Garbage Collector. There are no manual cleanup or `close` methods. Once all references are released, the object and its internal map are deallocated.

## Internal State & Concurrency
- **State:** The core state is a single public field: `Map<PrioritySlot, Integer> values`. This map is mutable and can be null.
    - **WARNING:** The public visibility of the `values` field is a deliberate performance-oriented design choice that bypasses getter/setter overhead. However, it breaks encapsulation. External code can directly modify the internal state, which can lead to unpredictable behavior if not managed carefully.

- **Thread Safety:** This class is **not thread-safe**. All operations on an instance are expected to occur on a single thread, typically a Netty I/O worker thread.
    - **WARNING:** Sharing an InteractionPriority instance across threads without external synchronization is unsafe and will lead to race conditions. If an instance must be passed from a network thread to a game logic thread, it should be treated as immutable or a deep copy should be created using the `clone` method.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static InteractionPriority | O(N) | Constructs an object by reading from a network buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided network buffer. Throws ProtocolException if constraints are violated. |
| computeSize() | int | O(1) | Calculates the number of bytes this object would occupy when serialized. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Reads a buffer to determine the byte size of a serialized object without full deserialization. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a structural integrity check on serialized data in a buffer. Does not validate semantic content. |
| clone() | InteractionPriority | O(N) | Creates a deep copy of the object, including a new HashMap instance for its values. |

*N represents the number of entries in the `values` map.*

## Integration Patterns

### Standard Usage
InteractionPriority is typically handled within the network layer. Game code will either receive it from a decoded packet or create one to be encoded into an outgoing packet.

```java
// Example: Decoding an object from a packet buffer
InteractionPriority priorityData = InteractionPriority.deserialize(packetBuffer, offset);
int bytesRead = InteractionPriority.computeBytesConsumed(packetBuffer, offset);

// Example: Creating an object to be sent
Map<PrioritySlot, Integer> priorities = new HashMap<>();
priorities.put(PrioritySlot.PRIMARY_ACTION, 100);
priorities.put(PrioritySlot.SECONDARY_ACTION, 50);

InteractionPriority outgoingPriority = new InteractionPriority(priorities);
packet.setInteractionData(outgoingPriority); // Assumes a packet structure
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Queuing:** Do not modify the public `values` map after the InteractionPriority object has been passed to the network layer for serialization. The serialization may happen on a different thread at a later time, leading to data races.
- **Using the Copy Constructor for Isolation:** The copy constructor (`new InteractionPriority(other)`) performs a shallow copy of the internal map. If true state isolation is required, use the `clone()` method to perform a deep copy.
- **Cross-Thread Sharing:** Never share a single instance between multiple threads without explicit locking. The non-atomic nature of map operations will lead to a corrupted state.

## Data Pipeline
The InteractionPriority object acts as a data record that flows through the network stack.

> **Incoming Flow:**
> Raw TCP Bytes -> Netty ByteBuf -> Protocol Decoder -> **InteractionPriority.deserialize()** -> Game Event -> Game Logic System

> **Outgoing Flow:**
> Game Logic System -> **new InteractionPriority()** -> Packet Encoder -> Netty ByteBuf -> Raw TCP Bytes

