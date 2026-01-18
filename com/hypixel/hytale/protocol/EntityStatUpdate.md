---
description: Architectural reference for EntityStatUpdate
---

# EntityStatUpdate

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class EntityStatUpdate {
```

## Architecture & Concepts
The EntityStatUpdate class is a fundamental Data Transfer Object (DTO) within the Hytale network protocol layer. It is not a service or a long-lived component; rather, it is a message that represents a discrete, atomic change to a gameplay-relevant statistic of an in-game entity, such as health, mana, or stamina.

Its primary architectural role is to serve as a structured container for data that must be efficiently and safely transmitted between the client and server. The class design is heavily optimized for performance and network bandwidth, employing a custom binary serialization format.

Key architectural features include:
*   **Custom Binary Layout:** Data is not serialized using a generic format. Instead, it follows a strict layout composed of a fixed-size block and a variable-size block. The fixed block contains direct values and offsets (pointers) into the variable block, allowing for fast, direct memory access within the buffer.
*   **Null Field Optimization:** A leading `nullBits` byte is used as a bitmask to indicate the presence or absence of nullable fields. This avoids wasting network bandwidth on empty optional data.
*   **Variable-Length Encoding:** Integer lengths for collections and strings are encoded using the VarInt format, which uses fewer bytes for smaller numbers.
*   **Client-Side Prediction Support:** The `predictable` boolean field is a critical flag for the client-side game engine. When true, it signals that the client can apply the stat change immediately for a smoother user experience, without waiting for authoritative confirmation from the server.

This class is the canonical representation of a stat change as it travels over the network.

### Lifecycle & Ownership
-   **Creation:** Instances are created under two distinct circumstances:
    1.  **On the sending endpoint (e.g., Server):** The game logic instantiates an EntityStatUpdate via its constructor when a stat change needs to be communicated. The object is populated and passed to the network serialization pipeline.
    2.  **On the receiving endpoint (e.g., Client):** The network protocol layer instantiates the object by invoking the static `deserialize` factory method, which reads data directly from an incoming Netty ByteBuf.

-   **Scope:** The lifecycle of an EntityStatUpdate instance is extremely short and tied to a single network packet processing event. It is created, serialized (or deserialized), processed, and then immediately becomes eligible for garbage collection.

-   **Destruction:** Ownership is transient. The object is not managed by a container or registry. It is automatically reclaimed by the Java Garbage Collector once the network processing or game logic handler that created it goes out of scope.

## Internal State & Concurrency
-   **State:** The class is fundamentally mutable. Its public fields are designed for high-performance access during the serialization and deserialization process. This design choice prioritizes raw speed over strict encapsulation, which is a common and acceptable trade-off for performance-critical DTOs in a game engine's network layer.

-   **Thread Safety:** **This class is not thread-safe.** Instances are designed to be created, manipulated, and read by a single thread, typically a Netty I/O worker thread or the main game loop thread. Concurrent modification from multiple threads will result in data corruption and unpredictable behavior. Any cross-thread usage must be managed with external synchronization, though such a pattern is strongly discouraged.

## API Surface
The public API is dominated by static methods for interacting with raw network buffers, reinforcing its role as a serialization-focused DTO.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static EntityStatUpdate | O(N) | Constructs an object by reading from a buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided buffer. Throws ProtocolException if data constraints are violated. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a pre-flight check on a buffer to ensure data is well-formed before attempting deserialization. Critical for security. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the total size of a serialized object within a buffer without performing a full deserialization. |
| clone() | EntityStatUpdate | O(N) | Creates a deep copy of the object, including all nested modifiers. |

*Complexity O(N) refers to the number of elements in the `modifiers` map and the length of the `modifierKey` string.*

## Integration Patterns

### Standard Usage
The class is used by game systems to create messages for the network layer, or by the network layer to deliver messages to game systems.

```java
// Example: Server-side creation and serialization
EntityStatUpdate healthUpdate = new EntityStatUpdate();
healthUpdate.op = EntityStatOp.Set;
healthUpdate.predictable = false; // Authoritative server state
healthUpdate.value = 85.0f;

// The network layer would then take this object and serialize it
// networkPipeline.send(healthUpdate);
```

```java
// Example: Client-side deserialization and processing
// ByteBuf received from network...
ValidationResult result = EntityStatUpdate.validateStructure(buffer, offset);
if (result.isOk()) {
    EntityStatUpdate update = EntityStatUpdate.deserialize(buffer, offset);
    // Game logic applies the update to the target entity
    player.getStats().applyUpdate(update);
}
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not modify and re-send the same EntityStatUpdate instance. These objects are cheap to create and re-using them can lead to subtle state-related bugs. Always create a new instance for each distinct message.
-   **Ignoring Validation:** Never call `deserialize` on a buffer received from a remote connection without first calling `validateStructure`. Bypassing validation exposes the application to denial-of-service attacks via malformed packets that could trigger exceptions or infinite loops.
-   **Cross-Thread Sharing:** Do not pass an instance of this class between threads without explicit and robust synchronization. It is intended for single-threaded access within the scope of a single task.

## Data Pipeline
The EntityStatUpdate class is a payload that moves through the network stack.

**Outbound (Sending) Flow:**
> Game Logic -> `new EntityStatUpdate()` -> Network Pipeline (`serialize`) -> Netty ByteBuf -> TCP/IP Stack

**Inbound (Receiving) Flow:**
> TCP/IP Stack -> Netty ByteBuf -> Protocol Dispatcher (`validateStructure` -> `deserialize`) -> **EntityStatUpdate Instance** -> Game Logic Handler -> Entity State Mutation

