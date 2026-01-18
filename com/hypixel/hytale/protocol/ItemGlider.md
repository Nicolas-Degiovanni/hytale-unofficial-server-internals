---
description: Architectural reference for ItemGlider
---

# ItemGlider

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class ItemGlider {
```

## Architecture & Concepts
The ItemGlider class is a **Protocol Data Structure**, not a service or manager. Its sole purpose is to serve as a structured, in-memory representation of a fixed-size data block that defines the physical properties of a glider item. It is a Plain Old Java Object (POJO) that directly maps to a 16-byte binary layout for network serialization and deserialization.

This class is a fundamental component of the low-level network protocol layer. It acts as a strongly-typed "struct" that provides an object-oriented view over a raw byte buffer, abstracting away the complexities of byte ordering and memory offsets. The use of Netty's ByteBuf and little-endian serialization methods confirms its role as a data container for network transport, ensuring consistent interpretation between the client and server.

It contains no business logic and is intended to be a transient data container used during the encoding and decoding phases of network communication.

## Lifecycle & Ownership
- **Creation:** An ItemGlider instance is created under two primary circumstances:
    1.  **Deserialization:** The static factory method `deserialize` is called by a network protocol decoder when processing an incoming packet. This is the most common creation path.
    2.  **Serialization:** Game logic instantiates a new ItemGlider and populates its fields before passing it to a protocol encoder to be written into an outgoing packet.
- **Scope:** **Transient**. The lifetime of an ItemGlider object is extremely short. It is designed to exist only for the duration of a single network packet's processing cycle. It should not be stored in long-lived game state objects.
- **Destruction:** The object becomes eligible for garbage collection as soon as the network handler or game logic that created it completes its operation. This typically happens immediately after its primitive data is copied into a more permanent game component, such as an entity's physics properties.

## Internal State & Concurrency
- **State:** The ItemGlider is a **fully mutable** data container. All of its fields are public primitives, allowing for direct and unrestricted modification. It does not maintain any internal cache or complex state beyond its four float properties.
- **Thread Safety:** This class is **not thread-safe**. Its public mutable fields make it inherently unsafe for concurrent access or modification. It is designed to be confined to a single thread, such as a Netty I/O worker thread or the main game loop thread.

**WARNING:** Sharing an ItemGlider instance across threads without external synchronization will lead to race conditions and unpredictable behavior. Such a pattern is a misuse of this class.

## API Surface
The public API is designed for efficient serialization and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static ItemGlider | O(1) | Factory method. Reads 16 bytes from the buffer at the given offset and constructs a new ItemGlider. |
| serialize(ByteBuf) | void | O(1) | Writes the object's 16 bytes of state into the provided buffer in little-endian format. |
| computeSize() | int | O(1) | Returns the constant size of the serialized data block, which is always 16. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a bounds check to ensure the buffer can contain a full 16-byte ItemGlider structure. |

## Integration Patterns

### Standard Usage
The canonical use case is within a network packet handler for decoding incoming data and applying it to the game state.

```java
// Example: Inside a network packet decoder or handler
public void processPacket(ByteBuf packetBuffer) {
    // Assume 'dataOffset' points to the start of the glider data
    int dataOffset = 12;

    ValidationResult result = ItemGlider.validateStructure(packetBuffer, dataOffset);
    if (!result.isOk()) {
        throw new ProtocolException(result.getErrorMessage());
    }

    ItemGlider gliderData = ItemGlider.deserialize(packetBuffer, dataOffset);

    // Immediately copy data to a persistent game component
    player.getPhysicsComponent().applyGliderProperties(gliderData);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store ItemGlider instances as fields in long-lived objects like an Entity or Player class. It is a transport object, not a state object. Copy its primitive values into the state object's own fields.
- **Manual Serialization/Deserialization:** Do not use `new ItemGlider()` and then manually call `buf.getFloatLE()` yourself. The static `deserialize` method is the correct and complete entry point for decoding.
- **Ignoring Validation:** Bypassing `validateStructure` before calling `deserialize` can lead to buffer under-read exceptions if the network packet is malformed or truncated.

## Data Pipeline
The ItemGlider serves as a data record that flows through the serialization and deserialization stages of the network stack.

> **Incoming Flow:**
> Raw Network Bytes -> Netty ByteBuf -> Protocol Decoder -> **ItemGlider.deserialize** -> Game State Update

> **Outgoing Flow:**
> Game State Change -> **new ItemGlider(...)** -> Protocol Encoder -> **ItemGlider.serialize** -> Netty ByteBuf -> Raw Network Bytes

