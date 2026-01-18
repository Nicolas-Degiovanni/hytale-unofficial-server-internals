---
description: Architectural reference for SelectedHitEntity
---

# SelectedHitEntity

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class SelectedHitEntity {
```

## Architecture & Concepts
The SelectedHitEntity class is a Plain Old Java Object (POJO) that functions as a Data Transfer Object (DTO) within the Hytale network protocol layer. Its primary responsibility is to encapsulate the state of a targeted or "hit" entity for transmission between the client and server.

Architecturally, this class represents a **fixed-size protocol message**. The entire structure is designed to occupy exactly 53 bytes on the wire, regardless of which optional fields are present. This design choice is critical for performance, as it allows the network layer to perform highly efficient, non-branching reads and to allocate buffers with predictable sizes.

To handle optional data within this fixed-size constraint, the class employs a bitmask field, colloquially referred to as *nullBits*. The first byte of the serialized data is a bitmask where each bit corresponds to a nullable field, indicating its presence or absence in the payload. If a field is absent (null), the serializer writes zero-padding for the space that field would have occupied, thus preserving the fixed 53-byte layout.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1.  **Inbound:** The network protocol decoder instantiates the object by calling the static `deserialize` method when a corresponding packet arrives from the network.
    2.  **Outbound:** Game logic instantiates the object using a constructor to represent a game event (e.g., a player's successful attack) before passing it to the network protocol encoder for serialization.

- **Scope:** Transient and extremely short-lived. An instance of SelectedHitEntity typically exists only for the duration of a single network packet's processing pipeline or a single game tick. It is a value object representing a point-in-time event.

- **Destruction:** The object is managed by the Java garbage collector. As it is designed for short-term use, it becomes eligible for collection as soon as it is no longer referenced by the network handler or game event processor. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** The class is a mutable data container. All fields are public and directly accessible, intended to be populated once upon creation and then read by a consumer system. It does not contain any internal caches or complex state logic.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, manipulated, and read within the confines of a single thread, such as a Netty I/O worker thread or the main game loop thread. Sharing instances across threads without external synchronization or deep cloning will lead to race conditions and undefined behavior.

## API Surface
The public contract is focused exclusively on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static SelectedHitEntity | O(1) | Constructs an object by reading 53 bytes from a buffer. This is the primary factory method. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state into a buffer, including the null-bitmask and any necessary padding. |
| computeSize() | int | O(1) | Returns the fixed size of the network structure, which is always 53. |
| clone() | SelectedHitEntity | O(1) | Performs a deep copy of the object and its contained value types. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if a buffer has enough readable bytes to contain this structure. |

## Integration Patterns

### Standard Usage
The object should always be created via deserialization when processing network data. It is a low-level protocol detail and should be abstracted away from high-level game logic, often by being wrapped in a more abstract GameEvent.

```java
// Example from within a network protocol decoder
public void decode(ChannelHandlerContext ctx, ByteBuf in) {
    // Validate that enough data exists for the entire structure
    if (ValidationResult.isError(SelectedHitEntity.validateStructure(in, in.readerIndex()))) {
        // Handle error: malformed packet
        return;
    }

    SelectedHitEntity hitData = SelectedHitEntity.deserialize(in, in.readerIndex());
    in.skipBytes(SelectedHitEntity.FIXED_BLOCK_SIZE);

    // Fire an event or pass the DTO to the game logic thread
    gameLogic.processEntityHit(hitData);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not maintain references to SelectedHitEntity instances beyond the immediate scope of processing. Storing this object in a cache or as part of a component's state is incorrect, as it represents a transient event, not persistent state.

- **Cross-Thread Sharing:** Never pass an instance from the network thread to the game thread without a concurrency-safe mechanism. The receiving system should either create a deep copy (using `clone`) or extract the primitive data into a new, thread-safe object.

- **Manual Field Serialization:** Do not attempt to read or write the fields to a buffer manually. The `serialize` and `deserialize` methods correctly handle the `nullBits` bitmask and padding, which is essential for protocol compliance. Bypassing these methods will create corrupted data.

## Data Pipeline
SelectedHitEntity serves as a data container that bridges the raw network stream with the game's event processing systems.

> Flow:
> Raw TCP Stream -> Netty ByteBuf -> Hytale Protocol Decoder -> **SelectedHitEntity Instance** -> Game Event Bus -> Combat System or Entity Interaction Logic

