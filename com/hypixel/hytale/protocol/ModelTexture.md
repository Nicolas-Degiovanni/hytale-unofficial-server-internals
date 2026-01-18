---
description: Architectural reference for ModelTexture
---

# ModelTexture

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO) / Transient

## Definition
```java
// Signature
public class ModelTexture {
```

## Architecture & Concepts
The ModelTexture class is a low-level data structure that represents a component of a larger game model within the Hytale network protocol. It is not a high-level service but a fundamental, serializable data container designed for extreme network efficiency.

Its primary architectural role is to define the precise binary layout for texture and weight information as it is transmitted between the client and server. The design explicitly avoids object overhead by using static methods for serialization and deserialization that operate directly on Netty ByteBuf instances. This pattern ensures zero-allocation reads and minimizes memory pressure during high-throughput packet processing.

Key concepts embodied in this class include:
- **Fixed-Size and Variable-Size Blocks:** The binary format is a hybrid. It starts with a fixed-size block (FIXED_BLOCK_SIZE) containing a null-bit field and the float weight. This is followed by a variable-size block for the optional texture string.
- **Null Bit Field:** A single byte at the start of the structure acts as a bitmask to indicate which nullable fields (in this case, the texture string) are present in the data stream. This is a common network optimization to avoid sending unnecessary length prefixes for absent data.
- **Direct Buffer Manipulation:** The class is tightly coupled with the Netty networking library, using ByteBuf directly for all I/O. This bypasses intermediate data structures and provides maximum performance.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1. **Deserialization:** The static deserialize method constructs a new ModelTexture instance by reading data directly from a network ByteBuf. This is the most common creation path for incoming data.
    2. **Direct Instantiation:** Game logic creates instances via the public constructor (e.g., new ModelTexture(...)) when preparing a model to be serialized and sent over the network.
- **Scope:** The lifetime of a ModelTexture object is extremely short and tied to the scope of a single network packet. It is a transient object, created for the immediate purpose of data transfer and then discarded.
- **Destruction:** The object is eligible for garbage collection as soon as the containing network packet has been fully processed or serialized. There are no long-lived references to ModelTexture instances.

## Internal State & Concurrency
- **State:** The class holds mutable state in its public fields, texture and weight. It is a simple data container with no internal logic for managing this state. It does not perform any caching.
- **Thread Safety:** **This class is not thread-safe.** It is a plain data object with no synchronization mechanisms. It is designed to be created, populated, and read within the confines of a single thread, typically a Netty I/O worker thread or a game logic thread. Concurrent modification from multiple threads will lead to race conditions and undefined behavior.

## API Surface
The public API is divided between instance methods for data manipulation and static methods for protocol-level I/O operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ModelTexture | O(N) | Constructs a new instance by reading from a ByteBuf. N is the length of the string. |
| serialize(buf) | void | O(N) | Writes the instance's state into the provided ByteBuf. N is the length of the string. |
| computeSize() | int | O(N) | Calculates the number of bytes required to serialize the instance. N is the length of the string. |
| computeBytesConsumed(buf, offset) | static int | O(1) | Calculates the size of a serialized object within a buffer without full deserialization. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a lightweight check on a buffer to validate if it contains a structurally correct object. |

## Integration Patterns

### Standard Usage
ModelTexture is almost never used in isolation. It is intended to be a component of a larger, more complex packet structure. The parent packet is responsible for invoking the serialization or deserialization logic.

```java
// Example: Deserializing from a network buffer within a parent packet
public void read(ByteBuf buffer) {
    // ... read other parent packet fields ...
    int textureOffset = buffer.readerIndex();
    this.modelTexture = ModelTexture.deserialize(buffer, textureOffset);
    buffer.readerIndex(textureOffset + ModelTexture.computeBytesConsumed(buffer, textureOffset));
    // ... read remaining fields ...
}

// Example: Preparing an object for serialization
ModelTexture textureToSend = new ModelTexture("hytale:stone", 1.0f);
ParentPacket packet = new ParentPacket();
packet.setModelTexture(textureToSend);
// The ParentPacket's serialize method would then call textureToSend.serialize(buf)
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not deserialize into an existing ModelTexture instance. The static deserialize method always returns a new object, and this pattern should be followed to avoid state corruption.
- **Multi-threaded Access:** Never share a ModelTexture instance across threads. Do not read from one thread while another is writing to it.
- **Ignoring Validation:** In security-sensitive contexts (e.g., server-side), failing to call validateStructure on untrusted input before attempting a full deserialize can expose the server to malformed packet exploits.

## Data Pipeline
The class acts as a marshalling/unmarshalling component in the network data pipeline. It translates between the in-memory Java object representation and the on-the-wire binary format.

> **Ingress (Client -> Server):**
> Netty ByteBuf -> **ModelTexture.deserialize** -> In-memory ModelTexture instance -> Game Logic

> **Egress (Server -> Client):**
> Game Logic -> In-memory ModelTexture instance -> **ModelTexture.serialize** -> Netty ByteBuf

