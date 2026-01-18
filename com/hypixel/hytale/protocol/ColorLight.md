---
description: Architectural reference for ColorLight
---

# ColorLight

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class ColorLight {
```

## Architecture & Concepts
The ColorLight class is a low-level, high-performance data structure representing a single colored light source within the Hytale protocol. It is not a high-level game object but rather a fundamental component of the network serialization layer, designed for efficiency and direct mapping to the on-wire data format.

This class acts as a "Packet Struct", a fixed-size data block that can be embedded within larger network packets. Its design, characterized by public fields and static serialization methods, prioritizes raw performance over encapsulation. This is a deliberate trade-off common in game networking to minimize object allocation overhead and CPU cycles during the critical path of packet processing.

The presence of static constants like FIXED_BLOCK_SIZE and MAX_SIZE indicates that ColorLight is designed to be consumed by a larger, automated protocol management system. This system likely uses these constants for pre-calculating packet sizes and buffer offsets without needing to instantiate the object itself, further optimizing network buffer management.

## Lifecycle & Ownership
- **Creation:** A ColorLight instance is created under two primary circumstances:
    1.  **Inbound:** By the network protocol layer when the static `deserialize` method is invoked on a raw Netty ByteBuf received from the network.
    2.  **Outbound:** By higher-level game systems (e.g., World, Rendering, or Entity systems) when preparing state to be transmitted to a remote peer.
- **Scope:** This object is **transient and short-lived**. Its lifetime should be confined to the scope of a single network packet's processing cycle. It is designed to be created, serialized or deserialized, and then immediately discarded.
- **Destruction:** The object is managed by the Java Garbage Collector. There are no native resources or manual cleanup procedures required. Due to its expected short lifespan, it is typically subject to fast, generational garbage collection.

## Internal State & Concurrency
- **State:** The object's state is entirely **mutable** and exposed through public fields. This is an intentional design choice for performance, eliminating the overhead of method calls for field access. The state consists of four bytes: radius, red, green, and blue.
- **Thread Safety:** ColorLight is **not thread-safe**. It contains no internal locking or synchronization primitives. It is designed to be operated on by a single thread at a time, such as a Netty I/O worker thread or the main game logic thread. Sharing a ColorLight instance across threads without external, user-managed synchronization will result in data corruption and undefined behavior.

## API Surface
The primary contract is direct field access and the static serialization/deserialization methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| radius, red, green, blue | byte | O(1) | Public fields for direct data manipulation. |
| deserialize(ByteBuf, int) | static ColorLight | O(1) | Reads 4 bytes from a buffer at a given offset and constructs a new ColorLight. |
| serialize(ByteBuf) | void | O(1) | Writes the 4 bytes of the object's state into the provided buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure a buffer contains enough data to deserialize an object. |
| computeSize() | int | O(1) | Returns the fixed size of the object on the wire, which is always 4. |

## Integration Patterns

### Standard Usage
The object is typically used as a data container when building or parsing a network packet. Developers should populate its fields and then pass it to a serializer, or receive it as the result of a deserializer.

```java
// Example: Preparing a light to be sent over the network
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;

// 1. Game logic creates and configures the light data
ColorLight lightData = new ColorLight();
lightData.radius = (byte) 8;
lightData.red = (byte) 255;
lightData.green = (byte) 150;
lightData.blue = (byte) 50;

// 2. The network system allocates a buffer and serializes the data
ByteBuf networkBuffer = Unpooled.buffer(4);
lightData.serialize(networkBuffer);

// networkBuffer is now ready for transmission
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not retain ColorLight instances as part of the persistent game state. They are transport objects. If you need to store light information, copy its values into a dedicated game-state class.
- **Cross-Thread Sharing:** Never pass a ColorLight instance from a network thread to a game logic thread (or vice-versa) without a proper synchronization or queuing mechanism. This will create severe race conditions.
- **Manual Serialization:** Do not attempt to read or write the four bytes manually. Always use the provided `serialize` and `deserialize` methods to ensure protocol compatibility and correctness.

## Data Pipeline
ColorLight serves as a data record that is moved between the game engine and the network stack.

**Outbound Flow (e.g., Placing a Torch):**
> Game Logic -> Creates **ColorLight** instance -> `serialize()` -> Netty ByteBuf -> Network Socket

**Inbound Flow (e.g., Receiving World Chunk Data):**
> Network Socket -> Netty ByteBuf -> `deserialize()` -> **ColorLight** instance -> World Renderer / Lighting Engine

