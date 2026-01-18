---
description: Architectural reference for BenchRequirement
---

# BenchRequirement

**Package:** com.hypixel.hytale.protocol
**Type:** Protocol Message / Transient

## Definition
```java
// Signature
public class BenchRequirement {
```

## Architecture & Concepts
The **BenchRequirement** class is a Data Transfer Object (DTO) that defines the binary wire format for crafting station requirements within the Hytale network protocol. It is not a service or manager, but rather a fundamental data contract that represents a piece of game state—specifically, the conditions a player must meet to use a particular "bench" like a crafting table or anvil.

Its primary architectural role is to serve as the serialization and deserialization boundary between raw network byte streams and the higher-level game logic. The class encapsulates a highly optimized, custom binary layout designed for performance and network efficiency.

This layout consists of two main parts:
1.  **Fixed-Size Block:** A 14-byte header containing a nullability bitmask, fixed-size fields (**type**, **requiredTierLevel**), and crucially, integer offsets to the variable-sized fields.
2.  **Variable-Size Block:** A contiguous data region following the fixed block, containing the actual data for nullable, variable-length fields like **id** and **categories**.

This "offset-based" serialization strategy is a key performance optimization. It allows consumers of the byte stream, such as the server, to read fixed-size data or validate the message structure without needing to parse the entire variable-length content. This is critical for defending against malformed packets and for efficiently routing data.

## Lifecycle & Ownership
- **Creation:** An instance of **BenchRequirement** is created in one of two scenarios:
    1.  **Deserialization:** The static factory method **deserialize** is invoked by the network protocol layer when an incoming packet is decoded from a Netty **ByteBuf**. This is the most common creation path on the receiving end.
    2.  **Direct Instantiation:** Game logic on the sending end (e.g., a server preparing to notify a client) creates an instance using its public constructor, populates its fields, and prepares it for serialization.

- **Scope:** Instances are ephemeral and short-lived. A **BenchRequirement** object typically exists only for the duration of processing a single network packet or a single game logic operation. It is created, used immediately, and then becomes eligible for garbage collection.

- **Destruction:** Managed entirely by the Java Garbage Collector. The class holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The class is fully **mutable**. All of its data fields are public and can be modified at any time after construction. This design facilitates easy construction and population of the object before it is passed to the serialization pipeline.

- **Thread Safety:** **BenchRequirement** is **not thread-safe** and must not be shared across threads without explicit, external synchronization. Its public mutable fields and stateful methods are designed for use within a single-threaded context, such as a Netty I/O worker thread or the main game loop.

**WARNING:** Concurrent modification of a **BenchRequirement** instance will lead to data corruption, race conditions, and unpredictable serialization output. Treat instances as thread-local.

## API Surface
The public API is dominated by static methods for interacting with the binary protocol format.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static BenchRequirement | O(N) | Constructs a new **BenchRequirement** by reading from a **ByteBuf**. Throws **ProtocolException** on malformed data. This is the primary entry point for the network read path. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided **ByteBuf** according to the defined binary format. This is the primary entry point for the network write path. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a non-allocating check of the binary data in a buffer to ensure it represents a valid object. **CRITICAL** for security to prevent DoS attacks from malformed packets. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will consume when serialized. Useful for pre-allocating buffers. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total size of a serialized object already present in a buffer by parsing its structure and variable-length fields. |

*N represents the total length of the variable-sized string and array data.*

## Integration Patterns

### Standard Usage
A typical use case involves either deserializing from a network buffer or creating an instance to be serialized.

**Deserialization (Receiving Data):**
```java
// In a network packet handler...
ByteBuf packetData = ...;

// First, validate the structure to ensure safety
ValidationResult result = BenchRequirement.validateStructure(packetData, 0);
if (!result.isOk()) {
    throw new ProtocolException("Invalid BenchRequirement: " + result.getReason());
}

// If valid, deserialize the object for game logic
BenchRequirement req = BenchRequirement.deserialize(packetData, 0);
gameLogic.processCraftingRequirement(req);
```

**Serialization (Sending Data):**
```java
// In game logic preparing a packet...
BenchRequirement req = new BenchRequirement();
req.type = BenchType.Crafting;
req.requiredTierLevel = 2;
req.id = "hytale:iron_anvil";
req.categories = new String[]{"weaponry", "tools"};

// Serialize the object into a buffer to be sent
ByteBuf out = Unpooled.buffer(req.computeSize());
req.serialize(out);
networkChannel.writeAndFlush(out);
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Validation:** Never call **deserialize** on a buffer received from an untrusted source without first calling **validateStructure**. Bypassing validation exposes the server to resource exhaustion and denial-of-service vulnerabilities from maliciously crafted packets (e.g., ones that declare an impossibly large array size).
- **Sharing Instances Across Threads:** Do not store a **BenchRequirement** instance in a shared context or pass it between threads. Its mutable nature makes it inherently unsafe for concurrent access. Each thread or task should work with its own private copy.
- **Modifying After Serialization:** Modifying a **BenchRequirement** object after it has been passed to a serialization pipeline can cause unpredictable behavior, as the serialization may happen on a different thread or at a later time. Treat objects as effectively immutable once they are scheduled for sending.

## Data Pipeline

The **BenchRequirement** class is a critical link in the network data pipeline, translating between raw bytes and structured game data.

**Inbound Data Flow (Read Path):**
> Flow:
> Raw TCP Stream -> Netty **ByteBuf** -> Protocol Packet Decoder -> **BenchRequirement.validateStructure()** -> **BenchRequirement.deserialize()** -> **BenchRequirement** Instance -> Game Logic System

**Outbound Data Flow (Write Path):**
> Flow:
> Game Logic System -> New **BenchRequirement** Instance -> **instance.serialize()** -> Protocol Packet Encoder -> Netty **ByteBuf** -> Raw TCP Stream

