---
description: Architectural reference for ItemReticle
---

# ItemReticle

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ItemReticle {
```

## Architecture & Concepts
The ItemReticle class is a Data Transfer Object (DTO) that defines the wire format for information about the player's targeting reticle. It serves as a strict data contract between the client and server, ensuring that both ends of the connection can correctly interpret the binary representation of reticle state.

This class is a fundamental component of the low-level Hytale network protocol. Its design prioritizes serialization performance and network efficiency over object-oriented encapsulation. This is evident in its public fields and static serialization methods that operate directly on Netty ByteBuf instances.

The binary layout is a hybrid of fixed-size and variable-size blocks:
1.  **Nullable Bit Field (1 byte):** A bitmask indicating the presence of optional fields. In this structure, it is used exclusively for the *parts* array.
2.  **Fixed-Size Block (6 bytes total):** Contains fields with a predictable size, allowing for fast, direct memory access. This includes the nullable bit field, the *hideBase* boolean, and the *duration* float.
3.  **Variable-Size Block:** Contains fields of dynamic length, such as the *parts* string array. Each element is prefixed with its length encoded as a VarInt to minimize byte usage for small values.

This structure is a common pattern in high-performance networking to reduce packet size and minimize CPU cycles during serialization and deserialization.

## Lifecycle & Ownership
-   **Creation:** An ItemReticle instance is created under two primary circumstances:
    1.  **Deserialization (Receiving):** The static factory method *deserialize* is invoked by the protocol decoder when an incoming network packet of the corresponding type is processed. The instance is constructed and populated directly from the raw bytes in a ByteBuf.
    2.  **Instantiation (Sending):** Game logic on the sending side creates a new instance via its constructor, populates its public fields, and passes it to the network layer for serialization.

-   **Scope:** The lifetime of an ItemReticle object is extremely short and ephemeral. It is designed to exist only for the immediate scope of packet processing. Once its data has been used to update game state (on receive) or has been written to a network buffer (on send), it is no longer needed and becomes eligible for garbage collection.

-   **Destruction:** Cleanup is handled automatically by the Java Garbage Collector. There are no native resources or manual memory management responsibilities associated with this class.

## Internal State & Concurrency
-   **State:** The class is fully mutable, with all data-holding fields declared as public. It is a simple data container and performs no internal caching or state transformation. Its state is a direct 1:1 representation of the data it holds.

-   **Thread Safety:** **This class is not thread-safe.** It contains no locks or other concurrency primitives. Instances are intended to be confined to a single thread, typically a Netty I/O worker thread or the main game loop thread.

    **WARNING:** Sharing an ItemReticle instance across multiple threads without external synchronization will result in data corruption and undefined behavior. Do not write to an instance from one thread while another thread is serializing or reading from it.

## API Surface
The primary contract of this class is its static serialization and validation interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | ItemReticle | O(N) | **[Factory]** Constructs an instance from a binary buffer. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the instance's state into the provided binary buffer according to the defined wire format. |
| computeSize() | int | O(M) | Calculates the number of bytes the object will consume when serialized. M is the number of strings in *parts*. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a read-only check of a buffer to ensure it contains a valid ItemReticle structure. Does not instantiate an object. |
| computeBytesConsumed(ByteBuf, int) | int | O(N) | Scans a buffer to determine the total size of a serialized ItemReticle without full deserialization. |

*N = Total byte size of all variable-length string data.*

## Integration Patterns

### Standard Usage
The class should be used as part of a larger packet-handling system. The sender creates and populates an instance, while the receiver uses the static methods to decode it from a buffer.

```java
// Example: Sending a reticle update
ItemReticle reticleUpdate = new ItemReticle();
reticleUpdate.hideBase = false;
reticleUpdate.duration = 2.5f;
reticleUpdate.parts = new String[]{"reticle_part_a", "reticle_part_b"};

// Assume 'packetBuffer' is a Netty ByteBuf being prepared for sending
reticleUpdate.serialize(packetBuffer);

// The buffer is now ready to be sent over the network
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not hold references to ItemReticle instances in long-lived collections or as member variables in game state managers. They represent a point-in-time snapshot and should be processed and discarded immediately.
-   **Cross-Thread Sharing:** Never pass an ItemReticle instance between threads. If data needs to be shared, extract the primitive values into a thread-safe structure or use an immutable copy.
-   **Ignoring ProtocolException:** The *deserialize* and *serialize* methods can throw ProtocolException. This is a critical error indicating data corruption or a client/server version mismatch. This exception must be caught and handled, typically by terminating the associated network connection.

## Data Pipeline
ItemReticle serves as a data model for a specific stage in the network communication pipeline.

> **Outbound Flow (Sending):**
> Game State Change -> `new ItemReticle(...)` -> **ItemReticle.serialize()** -> Netty Channel Pipeline -> Raw TCP/UDP Packet

> **Inbound Flow (Receiving):**
> Raw TCP/UDP Packet -> Netty Channel Pipeline -> ByteBuf -> **ItemReticle.deserialize()** -> Game State Update -> Render Engine

