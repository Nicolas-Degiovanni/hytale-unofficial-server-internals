---
description: Architectural reference for BenchUpgradeRequirement
---

# BenchUpgradeRequirement

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class BenchUpgradeRequirement {
```

## Architecture & Concepts
The BenchUpgradeRequirement class is a **Protocol Data Unit (PDU)**, also known as a Data Transfer Object (DTO). Its primary function is to model a specific, well-defined data structure within Hytale's binary network protocol. It is not a service or a manager; it is a passive container for data that is serialized for network transmission or deserialized from a network stream.

Architecturally, this class represents the cost—in materials and time—required to perform an in-game upgrade on a crafting station or a similar interactive block. It is a fundamental component of the game's progression and crafting systems.

The design is heavily optimized for low-latency network communication and minimal memory allocation overhead. It achieves this through a custom binary format rather than a more verbose format like JSON or XML. Key characteristics of its serialization strategy include:

*   **Hybrid Layout:** The binary structure is a mix of a fixed-size block and a variable-size block. The first 9 bytes are fixed (1 byte for a null-bitmask, 8 bytes for a double), allowing for predictable, high-speed reads.
*   **Null-Bitmasking:** A single leading byte acts as a bitmask to indicate the presence or absence of subsequent nullable, variable-length fields. This avoids the need for sentinel values or length prefixes when a field is null, saving bandwidth. In this class, the first bit of this byte controls the presence of the *material* array.
*   **Offset-Based Deserialization:** All static methods that read from a buffer, such as *deserialize* and *validateStructure*, operate on a Netty ByteBuf and an integer offset. This is a high-performance pattern that avoids creating sub-buffers or copying memory, allowing a single large buffer to be parsed sequentially for multiple distinct data structures.

## Lifecycle & Ownership
- **Creation:** Instances are created in two primary scenarios:
    1.  **Deserialization (Client/Server):** The static factory method *deserialize* is called by a higher-level network packet handler when parsing an incoming byte stream. This is the most common creation path.
    2.  **Manual Instantiation (Server):** Game logic on the server creates and populates an instance to define an upgrade's requirements before serializing it and sending it to a client.

- **Scope:** Transient and short-lived. An instance typically exists only for the duration of processing a single network packet or executing a single game-logic command. It is not designed to be cached or persist across sessions.

- **Destruction:** The object is managed by the Java Garbage Collector. Once all references to the instance are released (e.g., the packet handler method completes), it becomes eligible for garbage collection. There are no native resources or manual cleanup procedures associated with this class.

## Internal State & Concurrency
- **State:** The class is **highly mutable**. Its public fields, *material* and *timeSeconds*, can be directly accessed and modified after instantiation. It is a simple data holder and does not enforce any immutability constraints.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking, synchronization, or atomic operations.

    **WARNING:** Concurrent access from multiple threads is unsafe and will lead to race conditions, memory visibility issues, and undefined behavior. All interactions with an instance of BenchUpgradeRequirement must be confined to a single thread, such as a Netty I/O worker thread or the main game-tick thread. If data must be passed between threads, it should be done via a deep copy (using the *clone* method) or through a thread-safe queue.

## API Surface
The public API is designed for serialization, deserialization, and structural validation of network data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static BenchUpgradeRequirement | O(N) | Constructs an instance by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. N is the number of materials. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf. Throws ProtocolException if constraints are violated (e.g., array too long). |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the number of bytes this structure occupies in a buffer without full deserialization. Critical for advancing a buffer's read pointer. |
| computeSize() | int | O(N) | Calculates the number of bytes this object would occupy if serialized. Useful for pre-allocating buffers. |
| validateStructure(buffer, offset) | static ValidationResult | O(N) | Performs a read-only check to ensure the data at the given offset represents a valid structure. A crucial security measure to prevent parsing invalid or malicious packets. |
| clone() | BenchUpgradeRequirement | O(N) | Creates a deep copy of the object, including a new MaterialQuantity array with cloned elements. |

## Integration Patterns

### Standard Usage
A typical use case involves a network packet handler deserializing the structure from a buffer and passing it to the game logic for processing.

```java
// Within a packet handler on a Netty event loop...
public void handlePacket(ByteBuf packetData) {
    // Assume initial offset is known
    int offset = 4;

    // Pre-flight validation is critical on untrusted input
    ValidationResult result = BenchUpgradeRequirement.validateStructure(packetData, offset);
    if (!result.isValid()) {
        throw new ProtocolException("Invalid BenchUpgradeRequirement: " + result.error());
    }

    BenchUpgradeRequirement requirement = BenchUpgradeRequirement.deserialize(packetData, offset);

    // Pass the fully materialized object to the game engine
    game.processCraftingUpgrade(this.player, requirement);
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not deserialize into an existing BenchUpgradeRequirement instance or reuse a single instance for multiple outgoing messages. This can lead to subtle bugs where old state is not properly cleared. Always create a new instance for each distinct operation.
- **Ignoring Validation:** Never call *deserialize* on data from an untrusted source (like a game client) without first calling *validateStructure*. Bypassing this step can expose the server to Denial of Service (DoS) attacks, where a client sends a malformed packet with an extremely large array length, causing excessive memory allocation and a server crash.
- **Cross-Thread Modification:** Do not modify an instance on one thread while it is being serialized on another. For example, do not allow the main game thread to change the *material* array while a network thread is in the middle of calling the *serialize* method on it.

## Data Pipeline
The BenchUpgradeRequirement class serves as a data contract for moving crafting data between the server's game logic and the client's presentation layer.

**Outbound Flow (Server to Client):**
> Flow:
> Game Logic State -> **new BenchUpgradeRequirement()** -> serialize() -> Netty ByteBuf -> TCP/IP Stack

**Inbound Flow (Client to Server, if applicable):**
> Flow:
> TCP/IP Stack -> Netty ByteBuf -> validateStructure() -> **deserialize()** -> BenchUpgradeRequirement Instance -> Game Logic Processing

