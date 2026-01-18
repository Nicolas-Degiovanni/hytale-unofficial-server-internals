---
description: Architectural reference for BlockChange
---

# BlockChange

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class BlockChange {
```

## Architecture & Concepts
The BlockChange class is a fundamental Data Transfer Object (DTO) within the Hytale network protocol. It is not a service or manager, but rather a low-level data container representing a single, atomic modification to the game world's voxel grid.

Its primary architectural role is to serve as a highly efficient, serializable payload. The design prioritizes raw performance and minimal memory overhead for network transmission. This is achieved through a fixed-size binary layout, direct field access, and static methods for serialization and deserialization that operate directly on Netty ByteBuf instances. It represents the smallest unit of world state change that can be communicated between the server and client.

## Lifecycle & Ownership
- **Creation:** BlockChange instances are created under two primary circumstances:
    1. **Inbound:** The network layer instantiates it via the static `deserialize` method when parsing an incoming packet from a Netty ByteBuf.
    2. **Outbound:** The server-side game logic instantiates it using its constructor to encapsulate a world modification that must be broadcast to clients.
- **Scope:** The object's lifetime is exceptionally short and tied to the scope of a single network packet's processing cycle. It is created, used to update game state, and then immediately becomes eligible for garbage collection. It is not intended to be stored or referenced long-term.
- **Destruction:** Cleanup is managed entirely by the Java Garbage Collector. There are no native resources or explicit destruction methods.

## Internal State & Concurrency
- **State:** The state is fully mutable, with all fields exposed as public members. This design choice bypasses the overhead of getter and setter methods, which is critical in a high-throughput networking context where millions of these objects may be processed per second. The object holds no cached data and directly represents the state of a single block change.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data holder and contains no synchronization mechanisms. It is designed to be created, populated, and read within the confines of a single thread, typically a Netty I/O worker thread or the main game loop thread.

**WARNING:** Sharing a BlockChange instance across multiple threads without external locking will lead to race conditions and unpredictable world state corruption.

## API Surface
The public contract is focused exclusively on serialization, state representation, and basic object functionality.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static BlockChange | O(1) | Constructs a new BlockChange by reading a fixed 17 bytes from a buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state as 17 bytes into the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object, which is always 17. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a bounds check to ensure a buffer contains enough readable bytes for a valid object. |
| clone() | BlockChange | O(1) | Creates a shallow copy of the object. |

## Integration Patterns

### Standard Usage
BlockChange is almost never used in isolation. It is typically aggregated within a larger packet that represents a batch of world updates. The parent packet is responsible for managing the serialization and deserialization loop.

```java
// Example: Deserializing a list of changes from a parent packet
List<BlockChange> changes = new ArrayList<>();
int changeCount = buffer.readIntLE();
int offset = buffer.readerIndex();

for (int i = 0; i < changeCount; i++) {
    BlockChange change = BlockChange.deserialize(buffer, offset);
    changes.add(change);
    offset += BlockChange.FIXED_BLOCK_SIZE;
}

// Process the collected changes...
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not re-use a BlockChange instance for multiple distinct events. Its mutable nature makes this practice extremely error-prone. Always create a new instance or use `clone()` for each logical change.
- **Multi-threaded Modification:** Never modify a BlockChange instance from one thread while another thread is reading it or serializing it. This will result in partially written data and severe network protocol errors.
- **Partial Reads/Writes:** Do not attempt to read or write individual fields to a buffer. Always use the `serialize` and `deserialize` methods to ensure the correct byte order and layout are maintained.

## Data Pipeline
BlockChange is a critical link in the world state synchronization pipeline.

**Outbound (Server-Side):**
> Flow:
> Game Logic (e.g., player places block) -> **New BlockChange(x, y, z, id, rot)** -> Add to Packet -> Packet.serialize() calls **BlockChange.serialize()** -> Netty ByteBuf -> Network

**Inbound (Client-Side):**
> Flow:
> Network -> Netty ByteBuf -> Packet.deserialize() calls **BlockChange.deserialize()** -> **BlockChange instance** -> World Renderer / Client Game Logic -> Voxel Grid Updated

