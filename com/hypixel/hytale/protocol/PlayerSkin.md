---
description: Architectural reference for PlayerSkin
---

# PlayerSkin

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class PlayerSkin {
```

## Architecture & Concepts
The PlayerSkin class is a fundamental Data Transfer Object (DTO) within the Hytale network protocol, designed to represent a player's complete visual appearance. It is not a service or manager, but rather a structured data container that is serialized for network transmission and deserialized for use by the game client and server.

The core architectural pattern is a highly optimized custom binary format designed for performance and compactness. The serialization strategy avoids the overhead of generic formats like JSON or XML. It is composed of two main parts:

1.  **Fixed-Size Header Block (83 bytes):** This block contains metadata about the payload.
    *   **Nullability Bitfield (3 bytes):** A compact bitmask indicating which of the 20 possible string fields are present in the payload. This is a critical optimization, allowing optional fields to consume zero space if not set.
    *   **Offset Table (80 bytes):** A series of 20 4-byte integers. Each integer specifies the starting position of a corresponding string's data within the variable-size data block.

2.  **Variable-Size Data Block:** This block immediately follows the header and contains the raw, concatenated data for all non-null string fields. Each string is prefixed with a VarInt encoding its length.

This structure allows for extremely fast, non-allocating validation and offset calculation. A consumer of the data can immediately jump to any desired field without parsing the preceding ones. The strings themselves are asset identifiers, which are later resolved by the AssetManager to load the appropriate models and textures for rendering.

### Lifecycle & Ownership
-   **Creation:** PlayerSkin instances are created under two primary circumstances:
    1.  By the network protocol layer, which calls the static `deserialize` method to construct an object from an incoming network ByteBuf.
    2.  By high-level game logic, such as the character creation system or when a player's profile is loaded from storage. In these cases, the standard constructors are used.
-   **Scope:** This is a transient object with no persistent identity beyond its data. Its lifetime is bound to the component that holds it, such as a PlayerEntity or a network packet object. It is frequently created and destroyed.
-   **Destruction:** The object is managed by the Java garbage collector. There are no native resources or manual cleanup steps required.

## Internal State & Concurrency
-   **State:** The PlayerSkin is a mutable container of 20 nullable String fields. All fields are public, allowing for direct modification after instantiation. It holds no other internal state, performs no caching, and has no side effects.

-   **Thread Safety:** **This class is not thread-safe.** As a simple POJO, it contains no internal locking or synchronization mechanisms.

    **WARNING:** Concurrent reads and writes to a PlayerSkin instance will lead to race conditions and undefined behavior. If an instance must be shared between threads, synchronization must be handled externally by the calling code. The typical pattern is to deserialize on a network I/O thread and then safely hand off the immutable result to a game logic thread.

## API Surface
The public API is dominated by static methods for serialization, deserialization, and validation, reinforcing its role as a protocol-level data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static PlayerSkin | O(N) | Constructs a new PlayerSkin instance by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(K) | Performs a security-critical check on a buffer to ensure it contains a structurally valid PlayerSkin payload. Does not fully deserialize. |
| computeBytesConsumed(ByteBuf, int) | static int | O(K) | Calculates the total byte size of a serialized PlayerSkin within a buffer without deserializing it. Essential for packet framing. |
| computeSize() | int | O(K) | Calculates the total byte size the current object instance will occupy when serialized. |
| clone() | PlayerSkin | O(1) | Creates a shallow copy of the PlayerSkin object. The string references are copied, not the string data itself. |

*N = Total size in bytes of all string data. K = Number of non-null fields.*

## Integration Patterns

### Standard Usage
The most common use case is decoding a PlayerSkin from a network packet received by a Netty handler. Validation should always precede deserialization.

```java
// In a network handler, after receiving a ByteBuf
int offset = ...; // Start of PlayerSkin data in the buffer

ValidationResult result = PlayerSkin.validateStructure(buf, offset);
if (!result.isOk()) {
    // Disconnect client or log error
    throw new ProtocolException("Invalid PlayerSkin data: " + result.getReason());
}

PlayerSkin skin = PlayerSkin.deserialize(buf, offset);
playerEntity.setSkin(skin);
```

### Anti-Patterns (Do NOT do this)
-   **Skipping Validation:** Never call `deserialize` on data from an untrusted source (e.g., a game client) without first calling `validateStructure`. Failure to do so exposes the server to buffer overflow reads, denial-of-service attacks, and server crashes from malformed packets.
-   **Concurrent Modification:** Do not modify a PlayerSkin instance from one thread while it is being read or serialized by another. This will lead to data corruption. Pass copies or use proper synchronization.
-   **Miscalculating Offsets:** The `offset` parameter in static methods is critical. Passing an incorrect offset will cause deserialization to fail or, worse, read from an incorrect memory location within the buffer.

## Data Pipeline
PlayerSkin serves as the data payload for player appearance information as it moves between the client, server, and game engine subsystems.

> **Flow (Server to Client):**
> Player Entity Data -> **PlayerSkin instance** -> `serialize()` -> Network Packet -> Client Network Layer -> `validateStructure()` -> `deserialize()` -> **PlayerSkin instance** -> Render Component -> Asset Request

---

