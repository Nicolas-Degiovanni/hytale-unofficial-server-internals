---
description: Architectural reference for Cloud
---

# Cloud

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class Cloud {
```

## Architecture & Concepts
The Cloud class is a protocol-level Data Transfer Object that represents the visual properties of a cloud entity within the game world. It serves as a pure data container, defining the precise binary wire format for transmitting cloud appearance data between the client and server. This class is not a service or manager; its sole responsibility is the serialization and deserialization of its own state.

The binary layout is a custom, high-performance format designed to minimize network payload size. It consists of two main parts:
1.  **Fixed-Size Header (13 bytes):** This block begins with a 1-byte bitmask, `nullBits`, which indicates the presence or absence of the class's nullable fields. Following the bitmask are three 4-byte integer offsets.
2.  **Variable-Size Data Block:** This block contains the actual data for the fields that are present. The offsets in the header point to the starting position of each corresponding data element within this block.

This architectural pattern allows for efficient network transmission by completely omitting data for null fields and providing random access to serialized fields without parsing the entire object.

## Lifecycle & Ownership
- **Creation:** A Cloud instance is created under two primary circumstances:
    1.  By the network protocol layer via the static `deserialize` factory method when an incoming packet containing cloud data is being decoded.
    2.  By game logic via the `new Cloud()` constructor when defining or modifying a cloud's appearance before it is serialized and sent over the network.
- **Scope:** Instances are ephemeral and short-lived. Their lifetime is typically scoped to the processing of a single network packet or a single game-tick update. They are not designed for long-term storage or state management.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as all references to it are dropped, which typically occurs immediately after its data has been consumed by the rendering engine or written to an outbound network buffer.

## Internal State & Concurrency
- **State:** The Cloud object is highly mutable. Its public fields—`texture`, `speeds`, and `colors`—can be directly modified at any time after instantiation. It acts as a simple data structure with no internal encapsulation.

- **Thread Safety:** This class is **not thread-safe**. It contains no locks or other synchronization primitives. Concurrent access and modification from multiple threads will result in race conditions, inconsistent state, and undefined behavior.

    **Warning:** All interactions with a Cloud instance must be confined to a single thread, typically the Netty network event loop or the main game thread. Do not share instances across threads.

## API Surface
The public API is dominated by static utility methods for interacting with network buffers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static Cloud | O(N) | Constructs a new Cloud instance by decoding data from the given ByteBuf at a specific offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Encodes the instance's state into the given ByteBuf according to the defined binary format. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total size in bytes of a serialized Cloud object within a buffer without performing a full deserialization. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a pre-flight check on a buffer to verify structural integrity before attempting a full deserialization. |
| clone() | Cloud | O(N) | Creates a new Cloud instance. Note that this is not a full deep copy; the `speeds` map is copied by reference. |

## Integration Patterns

### Standard Usage
The canonical use case involves the network layer deserializing an object from a buffer, which is then passed to the game engine for processing.

```java
// In a network packet handler
// The buffer 'buf' and 'offset' are provided by the network layer.

if (Cloud.validateStructure(buf, offset).isOk()) {
    Cloud cloudData = Cloud.deserialize(buf, offset);
    
    // Pass the DTO to the rendering or game logic system
    worldRenderer.updateCloudAppearance(cloudData);
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Buffer Parsing:** Do not attempt to read the binary data from a ByteBuf manually. The format is complex, involving bitmasks and relative offsets. Always use the static `deserialize` method to ensure correctness.
- **Instance Reuse:** Do not reuse a Cloud instance to process data from multiple network packets. Its mutable state makes this pattern highly error-prone. Always create a new instance for each discrete message.
- **Cross-Thread Access:** Never modify a Cloud object from one thread while another thread is reading from it or serializing it. This will lead to data corruption.

## Data Pipeline
The Cloud class is a critical component in the data flow for world visual information. It acts as the bridge between the raw binary network stream and the structured in-memory representation used by the game engine.

> Flow (Inbound):
> Raw Network ByteBuf -> Protocol Decoder -> **Cloud.deserialize** -> In-Memory Cloud Instance -> Game State System / Rendering Engine

> Flow (Outbound):
> Game State Event -> `new Cloud(...)` -> **cloudInstance.serialize(buf)** -> Protocol Encoder -> Raw Network ByteBuf

