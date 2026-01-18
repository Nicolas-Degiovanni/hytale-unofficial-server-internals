---
description: Architectural reference for RangeVector3f
---

# RangeVector3f

**Package:** com.hypixel.hytale.protocol
**Type:** Data Structure / DTO

## Definition
```java
// Signature
public class RangeVector3f {
```

## Architecture & Concepts
The RangeVector3f class is a specialized, fixed-size data structure designed for high-performance network serialization. It is a fundamental building block within the Hytale network protocol, representing a three-dimensional vector where each axis (X, Y, Z) is defined by a floating-point range (a minimum and maximum value) rather than a single point.

Its primary architectural role is to serve as a Data Transfer Object (DTO) for network packets. This structure is commonly used to define volumes in game space, such as Axis-Aligned Bounding Boxes (AABBs), trigger zones, or query areas for entity lookups.

The design heavily prioritizes performance and predictability:
1.  **Fixed-Size Layout:** The class always occupies exactly 25 bytes when serialized. This eliminates the need for variable-length decoding, simplifying buffer management and reducing computational overhead on both the client and server.
2.  **Bitmask for Nullability:** A single leading byte acts as a bitmask (nullBits) to indicate which of the three component Rangef objects are present. This is a common network optimization that avoids the overhead of more complex serialization formats for optional fields while maintaining a fixed structure.
3.  **Direct Buffer Manipulation:** The static deserialize and instance serialize methods operate directly on Netty ByteBuf objects, integrating seamlessly into the engine's low-level networking pipeline.

## Lifecycle & Ownership
-   **Creation:** Instances are created under two primary circumstances:
    1.  **Inbound:** The network protocol layer instantiates RangeVector3f via the static `deserialize` method when decoding an incoming packet from a ByteBuf.
    2.  **Outbound:** Game logic instantiates it directly using its constructor when building a data structure that will be serialized into an outgoing network packet.
-   **Scope:** Transient and short-lived. A RangeVector3f object's lifetime is typically confined to the scope of a single network packet's processing cycle or a single game tick's logic. They are value objects, not long-lived entities.
-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as all references to it are dropped, which typically occurs immediately after the parent network packet has been fully processed or sent. No manual memory management is required.

## Internal State & Concurrency
-   **State:** Mutable. The public fields x, y, and z are directly accessible and can be modified after instantiation. The class is a simple data container with no internal logic for state management.
-   **Thread Safety:** **This class is not thread-safe.** It contains no locks or other synchronization primitives. It is designed to be created, manipulated, and read within a single thread, such as a Netty I/O thread or the main game logic thread.

    **WARNING:** Sharing a RangeVector3f instance across multiple threads without external, user-managed synchronization will result in race conditions and undefined behavior. Do not store instances in shared collections or pass them between threads without ensuring safe access patterns.

## API Surface
The public contract is defined by its serialization methods and direct field access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static RangeVector3f | O(1) | Constructs a new instance by reading a fixed 25-byte block from a ByteBuf. |
| serialize(buf) | void | O(1) | Writes the object's state into a fixed 25-byte block in the provided ByteBuf. |
| computeSize() | int | O(1) | Returns the constant serialized size of the object, which is always 25. |
| clone() | RangeVector3f | O(1) | Creates a deep copy of the object and its contained Rangef components. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a pre-check to ensure a buffer contains enough readable bytes for deserialization. |

## Integration Patterns

### Standard Usage
RangeVector3f is intended to be used as part of a larger network packet structure. It is either deserialized from a buffer or constructed manually before being serialized.

```java
// Example: Deserializing from a network buffer
ByteBuf networkBuffer = ...;
RangeVector3f receivedVolume = RangeVector3f.deserialize(networkBuffer, 0);
if (receivedVolume.x != null) {
    // Process the X range
}

// Example: Constructing and serializing for an outbound packet
Rangef xRange = new Rangef(10.0f, 20.0f);
Rangef zRange = new Rangef(5.0f, 15.0f);

// Y is null in this case
RangeVector3f queryVolume = new RangeVector3f(xRange, null, zRange);

ByteBuf outBuffer = Unpooled.buffer(queryVolume.computeSize());
queryVolume.serialize(outBuffer);
// outBuffer is now ready to be sent
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term State:** Do not retain instances of RangeVector3f in long-lived components or game state managers. They are transport objects, not stateful entities. Convert them to a more appropriate internal data structure if the data must persist.
-   **Cross-Thread Sharing:** Never pass a mutable RangeVector3f instance to another thread without a deep copy (using `clone`) or proper locking. The intended pattern is for one thread to own and operate on the object.
-   **Manual Serialization:** Do not attempt to read or write the 25-byte block manually. Always use the provided `serialize` and `deserialize` methods to ensure correctness, especially with the nullability bitmask.

## Data Pipeline
As a data structure, RangeVector3f does not process data itself; it *is* the data being processed. It represents a specific, structured payload within a larger network data flow.

> **Inbound Flow:**
> Raw TCP Socket -> Netty ByteBuf -> Protocol Decoder -> **RangeVector3f.deserialize** -> Game Logic

> **Outbound Flow:**
> Game Logic -> new **RangeVector3f()** -> Protocol Encoder -> **RangeVector3f.serialize** -> Netty ByteBuf -> Raw TCP Socket

