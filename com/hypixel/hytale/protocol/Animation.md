---
description: Architectural reference for the Animation network protocol object.
---

# Animation

**Package:** com.hypixel.hytale.protocol
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class Animation {
```

## Architecture & Concepts
The Animation class is a Data Transfer Object (DTO) specifically designed for high-performance network serialization. It is not a component of the runtime animation system itself, but rather a data-centric representation of an animation's state intended for transmission between the client and server.

Its core architectural feature is the implementation of a custom binary protocol. This protocol is optimized for both size and speed by separating data into two distinct regions within the serialized byte buffer:

1.  **Fixed-Size Block:** A 30-byte header containing primitive data types (floats, booleans, integers) and offsets for variable-sized data. This fixed layout allows for extremely fast, direct-memory access without parsing.
2.  **Variable-Size Block:** A subsequent data region containing dynamically sized fields like strings (name) and arrays (footstepIntervals).

A one-byte bitmask, the *nullable bit field*, is used at the beginning of the structure to efficiently encode the presence or absence of nullable, variable-sized fields. This avoids the overhead of writing null terminators or length prefixes for data that is not present.

This class acts as a self-contained serializer and deserializer for its own data structure, bridging the gap between the high-level game state and the low-level network byte stream.

### Lifecycle & Ownership
-   **Creation:** Instances are created under two primary circumstances:
    1.  By the network layer calling the static `deserialize` method to construct an object from an incoming `ByteBuf`.
    2.  By game logic using the constructor (`new Animation(...)`) to define an animation that needs to be sent over the network.
-   **Scope:** The object's lifetime is typically very short. It exists only as a transient container to be either serialized into a buffer or used immediately after being deserialized from one. It is not intended to be a long-lived stateful object.
-   **Destruction:** The object is managed by the Java Garbage Collector and is eligible for cleanup as soon as all references to it are dropped. No manual resource management is required.

## Internal State & Concurrency
-   **State:** The state is fully mutable. All fields are public, prioritizing raw performance and ease of access over encapsulation. This is a deliberate design choice for a DTO in a performance-critical path. The object holds no external resources or handles.
-   **Thread Safety:** This class is **not thread-safe**. All fields are publicly accessible without any synchronization mechanisms. Concurrent reads and writes will lead to race conditions and undefined behavior.

    **WARNING:** An Animation instance must only be accessed and modified by a single thread at any given time. If it must be passed between threads (e.g., from a network thread to a game logic thread), it should be done through a thread-safe queue, and ownership must be clearly transferred. Do not maintain shared references to an instance across threads.

## API Surface
The primary contract of this class is its serialization and validation logic, not traditional methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided Netty ByteBuf according to the custom binary protocol. N is the size of variable fields. |
| deserialize(ByteBuf, int) | static Animation | O(N) | Decodes a new Animation object from the given ByteBuf at the specified offset. Throws ProtocolException on malformed data. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs cheap, non-allocating checks on a buffer to validate offsets and lengths. Critical for server-side security to prevent malformed packet exploits. |
| computeSize() | int | O(1) | Calculates the exact number of bytes required to serialize the current object state. Useful for pre-allocating buffers. |
| clone() | Animation | O(N) | Creates a deep copy of the footstepIntervals array and a shallow copy of other fields. |

## Integration Patterns

### Standard Usage
The class is intended to be used by network packet handlers for encoding and decoding.

**Serialization (Outbound Packet)**
```java
// Game logic creates an animation to be sent
Animation anim = new Animation(
    "player_run", 1.0f, 0.2f, true, 1.0f, new int[]{10, 25}, 123, 0
);

// The network layer serializes it into a buffer
ByteBuf buffer = Unpooled.buffer();
anim.serialize(buffer);
networkManager.send(buffer);
```

**Deserialization (Inbound Packet)**
```java
// Network handler receives a buffer
ByteBuf incomingBuffer = ...;

// Validate before processing to prevent exploits
ValidationResult result = Animation.validateStructure(incomingBuffer, 0);
if (!result.isOk()) {
    throw new ProtocolException("Invalid Animation structure: " + result.getMessage());
}

// Deserialize into an object
Animation receivedAnim = Animation.deserialize(incomingBuffer, 0);
gameLogic.processAnimation(receivedAnim);
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not hold references to Animation objects as part of a long-lived component's state. They are designed as ephemeral data carriers, not stateful components. Clone the data into a more stable, encapsulated domain object if persistence is needed.
-   **Concurrent Modification:** Never modify an Animation object from one thread while another thread might be reading it or serializing it. This will lead to data corruption in the serialized payload.
-   **Ignoring Validation:** On the server, never call `deserialize` on a buffer received from a client without first calling `validateStructure`. A malicious client could send invalid offsets or lengths, causing the deserializer to throw exceptions or attempt out-of-bounds memory access.

## Data Pipeline
The Animation class is a key link in the network data pipeline, converting object-oriented data into a binary stream and back.

> **Outbound Flow:**
> Game Logic → `new Animation()` → **Animation.serialize()** → Netty `ByteBuf` → Network Channel

> **Inbound Flow:**
> Network Channel → Netty `ByteBuf` → **Animation.validateStructure()** → **Animation.deserialize()** → Game Logic

