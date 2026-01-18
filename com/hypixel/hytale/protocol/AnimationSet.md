---
description: Architectural reference for AnimationSet
---

# AnimationSet

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class AnimationSet {
```

## Architecture & Concepts
The AnimationSet class is a protocol-level Data Transfer Object (DTO). It is not a service or a manager; its sole responsibility is to represent a structured, serializable collection of animation data for network transmission. It serves as a fundamental building block within the Hytale network protocol, defining the wire format for how animation sets are sent between the server and client.

The binary layout of a serialized AnimationSet is highly optimized for performance and minimal size, employing two key strategies:

1.  **Nullable Bit Field:** The first byte of the serialized data is a bitmask. Each bit corresponds to a nullable field within the class (id, animations, nextAnimationDelay). If a bit is set, the corresponding data is present in the stream. This avoids wasting bandwidth on null or default values.

2.  **Offset-Based Variable Data:** The structure is divided into a fixed-size block and a variable-size block. The fixed block contains metadata and, critically, integer offsets that point to the location of variable-length data (such as the id string and the animations array) within the variable block. This allows for extremely fast parsing and the ability to calculate the total size of the object without reading its entire contents.

This class, along with its dependencies like Animation and Rangef, forms a self-contained serialization and deserialization unit. It operates directly on Netty ByteBuf instances, making it a low-level component tightly coupled to the networking layer.

## Lifecycle & Ownership
-   **Creation:** An AnimationSet instance is created under two distinct circumstances:
    1.  **On the sending endpoint (e.g., Server):** Instantiated via its constructor (`new AnimationSet(...)`) and populated with data from the game state. This instance is short-lived and exists only to be passed to the `serialize` method.
    2.  **On the receiving endpoint (e.g., Client):** Instantiated exclusively by the static factory method `deserialize`. It is a critical design principle that receivers **never** use the `new` keyword for this class.

-   **Scope:** The lifetime of an AnimationSet object is exceptionally brief. It is scoped to the processing of a single network packet. Once deserialized, its data is typically copied into higher-level game engine structures (e.g., an entity's animation controller), and the AnimationSet object is immediately eligible for garbage collection.

-   **Destruction:** The object is managed by the Java garbage collector. There are no manual cleanup or `close` methods. Due to its transient nature, it is reclaimed quickly after its data has been consumed by the game logic.

## Internal State & Concurrency
-   **State:** The class is a **mutable** data container with public fields. This design choice prioritizes performance by avoiding the overhead of getters and setters during the high-frequency operations of serialization and deserialization. However, once an instance is created (either programmatically or via deserialization), it should be treated as immutable by consuming systems.

-   **Thread Safety:** This class is **not thread-safe**. All methods that interact with a ByteBuf (`serialize`, `deserialize`, `validateStructure`, `computeBytesConsumed`) are fundamentally unsafe to call from multiple threads on the same buffer without external synchronization. It is designed to be used within a single-threaded context, such as a Netty event loop or the main game thread.

## API Surface
The public contract is dominated by static methods for interacting with raw byte buffers, reinforcing its role as a protocol utility.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static AnimationSet | O(N) | Constructs an AnimationSet by reading from a buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the instance's state into the provided buffer according to the defined wire format. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a security and integrity check on the buffer data without full deserialization. CRITICAL for preventing crashes from malformed packets. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the total size in bytes that the object occupies in the buffer. Faster than a full deserialize. |
| computeSize() | int | O(M) | Calculates the required buffer size for serializing the current instance, where M is the number of animations. |

## Integration Patterns

### Standard Usage
The canonical use case involves a network packet handler decoding a buffer. The `validateStructure` method should always be called before attempting to deserialize.

```java
// In a network handler, after identifying the packet type
ValidationResult result = AnimationSet.validateStructure(packetBuffer, offset);
if (!result.isValid()) {
    // Disconnect client or log error; do not proceed
    throw new ProtocolException("Invalid AnimationSet structure: " + result.error());
}

AnimationSet set = AnimationSet.deserialize(packetBuffer, offset);
int bytesRead = AnimationSet.computeBytesConsumed(packetBuffer, offset);

// Pass the 'set' object to the animation system
game.getAnimationSystem().processNewSet(entityId, set);
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Validation:** Never call `deserialize` without first calling `validateStructure`. A malicious or corrupted packet could provide invalid offsets or lengths, leading to buffer overflows, exceptions, and potential server instability.
-   **Modifying Deserialized Objects:** Do not alter the fields of an AnimationSet instance received from the network. Treat it as a read-only snapshot of data. If modifications are needed, clone it or copy its data to a different domain object.
-   **Shared State:** Do not retain references to AnimationSet objects long-term or share them between systems. Their purpose is to be consumed immediately and then discarded.

## Data Pipeline
The AnimationSet acts as a data contract for a specific stage in the network pipeline. Its journey is unidirectional and ephemeral.

> Flow:
> Server Game State -> **AnimationSet (Instance)** -> `serialize()` -> Netty ByteBuf -> Network -> Client ByteBuf -> `deserialize()` -> **AnimationSet (New Instance)** -> Client Animation Controller

