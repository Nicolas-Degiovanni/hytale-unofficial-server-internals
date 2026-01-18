---
description: Architectural reference for EntityStatEffects
---

# EntityStatEffects

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class EntityStatEffects {
```

## Architecture & Concepts
The EntityStatEffects class is a protocol-level Data Transfer Object (DTO) that defines the binary representation for visual and audio effects associated with an entity's status. It is not a service or manager, but rather a structured *message format* used for network communication between the client and server.

Its primary architectural function is to serve as a strict, low-level contract for serialization and deserialization. The design is heavily optimized for network performance and memory efficiency, employing a hybrid fixed-block and variable-block layout.

-   **Fixed Block:** The first 6 bytes of the serialized object contain predictable, fixed-size fields (triggerAtZero, soundEventIndex). This allows for rapid, direct-offset reads.
-   **Nullable Bit Field:** The very first byte is a bitmask used to encode the presence or absence of subsequent nullable, variable-sized fields. This is a common pattern in high-performance binary protocols to avoid wasting bytes on null pointers or empty collections. In this class, it controls the serialization of the *particles* array.
-   **Variable Block:** Following the fixed block, a variable-length section contains the optional array of ModelParticle objects. The length of this array is prefixed with a VarInt to minimize byte usage for small collections.

This class is a foundational element of the rendering and audio feedback systems, translating abstract game events into concrete data that can be transmitted over the network and interpreted by the client.

## Lifecycle & Ownership
-   **Creation:**
    -   **Receiving End:** Instantiated exclusively by a higher-level protocol decoder via the static `deserialize` factory method when processing an incoming network packet.
    -   **Sending End:** Instantiated directly via `new EntityStatEffects(...)` by server-side game logic that needs to communicate a status effect to a client.
-   **Scope:** Extremely short-lived and transient. An instance of EntityStatEffects typically exists only for the duration of a single network packet's processing pipeline. It is not designed to be cached or held in long-term state.
-   **Destruction:** The object becomes eligible for garbage collection immediately after its data has been consumed by the game logic (on the receiving end) or after it has been written to a network buffer (on the sending end). Ownership is simple and ephemeral.

## Internal State & Concurrency
-   **State:** Mutable. All fields are public and directly accessible. The class is intended to be populated once and then immediately serialized, or deserialized once and then immediately read. It performs no internal caching.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded access, typically within a Netty I/O worker thread or the main game loop. All serialization and deserialization methods operate on a Netty ByteBuf, which has its own strict threading model. Unsynchronized, concurrent access from multiple threads will result in data corruption and undefined behavior. External locking is required if multi-threaded access is unavoidable.

## API Surface
The public contract is dominated by static methods for serialization, deserialization, and validation, reflecting its role as a data codec.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | EntityStatEffects | O(N) | **[Factory]** Constructs an object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf. Throws ProtocolException if constraints are violated (e.g., array too long). |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total byte size of a serialized object within a buffer without allocating a new object. Crucial for advancing buffer read pointers. |
| computeSize() | int | O(N) | Calculates the byte size this object will occupy when serialized. Used for pre-allocating buffers. |
| validateStructure(buffer, offset) | ValidationResult | O(N) | Performs a read-only check of the byte stream to ensure structural integrity before attempting a full deserialization. **Critical for security and stability.** |

*N represents the number of elements in the particles array.*

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol layer. On the receiving end, a packet handler deserializes the object from a buffer to apply its effects.

```java
// Example within a network packet handler
public void handlePacket(ByteBuf packetData) {
    // Validate structure on untrusted input before processing
    ValidationResult result = EntityStatEffects.validateStructure(packetData, 0);
    if (!result.isValid()) {
        throw new MalformedPacketException(result.error());
    }

    // Deserialize and consume the data
    EntityStatEffects effects = EntityStatEffects.deserialize(packetData, 0);
    gameWorld.getEffectSystem().applyEffects(effects);
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not retain instances of EntityStatEffects in game state components or caches. They are transient network messages. If the data must be preserved, map it to a dedicated, long-lived game state object.
-   **Ignoring Validation:** Never call `deserialize` on a buffer received from an untrusted source (like a game client) without first calling `validateStructure`. Bypassing this step can lead to `OutOfMemoryError` or other exceptions if a malicious client sends an invalid array length, creating a denial-of-service vulnerability.
-   **Cross-Thread Sharing:** Do not pass an instance of this class between threads without proper synchronization. For example, do not allow a network thread to deserialize into an object while a game logic thread is simultaneously reading from it.

## Data Pipeline
EntityStatEffects acts as a data container that flows from game logic into the network on the sending side, and from the network into game logic on the receiving side.

> **Sending Flow:**
> Game Event -> **new EntityStatEffects()** -> serialize() -> Netty ByteBuf -> Network Driver

> **Receiving Flow:**
> Network Driver -> Netty ByteBuf -> validateStructure() -> **deserialize()** -> Game Logic -> Audio/Visual Systems

