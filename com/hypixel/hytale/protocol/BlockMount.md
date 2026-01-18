---
description: Architectural reference for BlockMount
---

# BlockMount

**Package:** com.hypixel.hytale.protocol
**Type:** Data Structure / DTO

## Definition
```java
// Signature
public class BlockMount {
```

## Architecture & Concepts
The BlockMount class is a specialized Data Transfer Object (DTO) designed for high-performance network communication and data serialization. It represents a single, mountable point on a game block, such as a seat on a chair or a handle on a door. Its primary role is to serve as a strict, language-neutral contract for how mount point data is structured in binary form.

This class is a fundamental component of the Hytale protocol layer. Its design prioritizes raw performance and predictable memory layout over object-oriented encapsulation. The entire structure is engineered to map directly to a fixed-size, 30-byte block of memory, which is critical for efficient buffer processing in the Netty-based network stack. It is not a service or a manager; it is pure, structured data.

A key architectural feature is its explicit handling of nullable fields. Instead of relying on language-level nulls that have no direct binary representation, it employs a bitmask (the `nullBits` byte) to encode the presence or absence of the `position` and `orientation` vectors. This is a common low-level optimization to maintain a fixed-size structure while supporting optional data fields.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand. The primary creation pathways are the static `deserialize` method, which constructs an object from a network `ByteBuf`, or direct instantiation by systems that programmatically define block behaviors. It is not managed by a dependency injection framework.
- **Scope:** The object's lifetime is transient and strictly bound to its container. For example, a BlockMount instance created during packet deserialization exists only as long as the parent packet object is in scope. It is intended for short-lived, immediate use.
- **Destruction:** The object is managed by the Java Garbage Collector. There are no manual cleanup or `close` methods. It is eligible for garbage collection as soon as all references to it are dropped.

## Internal State & Concurrency
- **State:** The internal state is fully **mutable**. All fields, including `type`, `position`, `orientation`, and `blockTypeId`, are public and can be modified directly at any time. The class acts as a simple data container with no internal logic to protect its state.
- **Thread Safety:** This class is **not thread-safe**. Its mutable nature makes it inherently unsafe for concurrent access. If an instance is shared between threads, the caller is responsible for implementing external synchronization (e.g., locks). It is designed to be created, populated, and read within a single-threaded context, such as a Netty event loop thread or the main game thread.

## API Surface
The public API is focused exclusively on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static BlockMount | O(1) | Constructs a BlockMount by reading a fixed 30-byte block from a ByteBuf. |
| serialize(buf) | void | O(1) | Writes the object's state into a fixed 30-byte block in the provided ByteBuf. |
| computeSize() | int | O(1) | Returns the constant size of the binary representation (30). |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a pre-check to ensure a buffer has enough readable bytes for deserialization. |
| clone() | BlockMount | O(1) | Creates a deep copy of the instance and its contained Vector3f objects. |

## Integration Patterns

### Standard Usage
The primary use case for BlockMount is within the network protocol and asset loading systems. A developer would typically receive this object from a higher-level API rather than constructing it manually.

```java
// Example: Deserializing from a network buffer
// This logic is typically encapsulated within a packet class.

ByteBuf networkBuffer = ...;
ValidationResult result = BlockMount.validateStructure(networkBuffer, 0);

if (result.isOk()) {
    BlockMount mountPoint = BlockMount.deserialize(networkBuffer, 0);
    // Use the deserialized mountPoint object in game logic
    System.out.println("Mount type: " + mountPoint.type);
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Do not share a BlockMount instance across threads without external locking. Modifying its public fields from one thread while another reads them will lead to data corruption and unpredictable behavior.
- **Ignoring Nullability:** The `position` and `orientation` fields can be null. Always perform a null check before attempting to access their methods. The binary format guarantees this contract via the `nullBits` field, but the Java object does not.
- **Assuming Variable Size:** Do not attempt to write more or less than 30 bytes during custom serialization. The entire protocol relies on this class having a fixed, predictable size.

## Data Pipeline
BlockMount acts as a data model that is passed through serialization and deserialization pipelines. It does not initiate or manage data flow itself.

> **Inbound Flow (Network to Game):**
> Netty ByteBuf -> Protocol Packet Decoder -> **BlockMount.deserialize** -> In-Memory BlockMount Object -> Block Definition Registry

> **Outbound Flow (Game to Network):**
> Block Definition -> In-Memory BlockMount Object -> **blockMount.serialize** -> Protocol Packet Encoder -> Netty ByteBuf

