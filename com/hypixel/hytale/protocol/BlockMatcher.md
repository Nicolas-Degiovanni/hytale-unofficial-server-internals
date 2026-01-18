---
description: Architectural reference for BlockMatcher
---

# BlockMatcher

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class BlockMatcher {
```

## Architecture & Concepts
The BlockMatcher class is a fundamental data transfer object (DTO) within the Hytale network protocol. It does not represent a concrete block in the world but rather a **declarative rule** used to identify or match one or more blocks based on specific criteria. Its primary function is to provide a compact, serializable representation of a block-matching query.

This component is critical for any system that needs to communicate world-state conditions between the client and server. Examples include:
- **Quest Objectives:** Defining a condition like "place any type of log block on a grass face".
- **Building Systems:** Validating if a structure is built correctly according to a blueprint.
- **Procedural Generation:** Passing rules for structure or feature placement.

The structure is optimized for binary serialization. It uses a bitfield (`nullBits`) to efficiently encode the presence of optional, variable-sized child objects like BlockIdMatcher, minimizing payload size over the network.

## Lifecycle & Ownership
- **Creation:** A BlockMatcher instance is created under two primary circumstances:
    1.  **Deserialization:** The static `deserialize` method constructs an instance from an incoming Netty ByteBuf. This is the most common path for data received from the network.
    2.  **Direct Instantiation:** Game logic on the client or server can instantiate a BlockMatcher using `new BlockMatcher()` to define a new rule before serializing it for transmission.

- **Scope:** The object's lifetime is typically very short and bound to the scope of a single operation. For network data, it exists only for the duration of packet processing within a Netty pipeline handler. For game logic, it exists until it has been serialized or the logic that created it completes.

- **Destruction:** The object is managed by the Java Garbage Collector. There are no native resources or explicit cleanup methods. It is reclaimed once it falls out of scope and no longer has any active references.

## Internal State & Concurrency
- **State:** The BlockMatcher is a **mutable** data container. Its public fields, such as block and face, can be modified directly after instantiation. It does not maintain any internal cache or complex state beyond its immediate fields.

- **Thread Safety:** This class is **not thread-safe**. It is a simple POJO with no internal locking or synchronization mechanisms. It is designed to be created, populated, and read within a single-threaded context, such as a Netty I/O thread or the main game update loop.

    **WARNING:** Concurrent modification of a BlockMatcher instance from multiple threads will lead to race conditions and unpredictable behavior. All access must be externally synchronized if multi-threaded use is unavoidable.

## API Surface
The public API is dominated by static methods for serialization and validation, reinforcing its role as a protocol-level component.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static BlockMatcher | O(N) | Constructs a new BlockMatcher by reading from a ByteBuf at a given offset. N depends on the size of the nested BlockIdMatcher. |
| serialize(buf) | void | O(N) | Writes the current state of the object into the provided ByteBuf. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the number of bytes a serialized BlockMatcher occupies in a buffer without full deserialization. |
| computeSize() | int | O(N) | Calculates the number of bytes required to serialize the current object state. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a structural integrity check on serialized data in a buffer without creating an object. Returns OK or an error. |
| clone() | BlockMatcher | O(N) | Creates a deep copy of the BlockMatcher and its nested objects. |

## Integration Patterns

### Standard Usage
BlockMatcher is almost always used as part of a larger data structure being sent over the network. It is rarely used in isolation.

**Scenario 1: Deserializing from a Network Packet**
```java
// Inside a network handler, reading from a ByteBuf
public void read(ByteBuf buffer) {
    // ... read other packet fields ...
    int currentOffset = ...;
    BlockMatcher condition = BlockMatcher.deserialize(buffer, currentOffset);
    int bytesRead = BlockMatcher.computeBytesConsumed(buffer, currentOffset);
    currentOffset += bytesRead;
    // ... use the 'condition' object in game logic ...
}
```

**Scenario 2: Serializing for Network Transmission**
```java
// In game logic, preparing a packet to send
BlockIdMatcher idMatcher = new BlockIdMatcher(BlockId.STONE);
BlockMatcher condition = new BlockMatcher(idMatcher, BlockFace.Up, false);

// ... get the packet's ByteBuf ...
ByteBuf buffer = getPacketBuffer();
condition.serialize(buffer);
// ... send the packet ...
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not reuse a BlockMatcher instance across different logical operations by modifying its fields. This can lead to subtle bugs. It is safer and clearer to create a new instance for each new rule.
- **Concurrent Access:** Never share a BlockMatcher instance between threads without explicit locking. For example, do not read its properties on a render thread while it is being deserialized on a network thread.

## Data Pipeline
BlockMatcher acts as a translation layer between a raw byte stream and a structured, in-memory representation of a game rule.

> **Inbound Flow (Server to Client):**
> Network ByteBuf -> Protocol Decoder -> **BlockMatcher.deserialize** -> In-Memory BlockMatcher -> Game Logic (e.g., Quest System)

> **Outbound Flow (Client to Server):**
> Game Logic (e.g., Build Tool) -> In-Memory BlockMatcher -> **BlockMatcher.serialize** -> Protocol Encoder -> Network ByteBuf

