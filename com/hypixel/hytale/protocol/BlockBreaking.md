---
description: Architectural reference for BlockBreaking
---

# BlockBreaking

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class BlockBreaking {
```

## Architecture & Concepts

The BlockBreaking class is a **Protocol Data Unit (PDU)**, a specialized Data Transfer Object designed for network serialization. Its primary function is to represent the state of a block being damaged or mined within the game world. It serves as a structured container for data that is exchanged between the client and server.

This class is not a component of the core game logic; it contains no behavioral methods. Instead, it acts as a data contract, defining the precise binary layout for block-breaking information. Its design is heavily optimized for network efficiency, employing a custom serialization format that minimizes payload size.

The binary structure consists of two main parts:
1.  A **fixed-size block** of 25 bytes that contains primitive data types and offsets to variable-length fields.
2.  A **variable-size block** appended after the fixed block, which stores string data.

A single-byte bitmask, `nullBits`, is used at the beginning of the structure to efficiently encode the presence or absence of nullable fields, preventing the need to transmit empty strings or sentinel values. This design is critical for maintaining low-latency communication in a real-time environment.

## Lifecycle & Ownership

-   **Creation:**
    -   **Inbound (Deserialization):** Instances are created exclusively by the static `deserialize` factory method. This method is invoked by a higher-level network protocol decoder, such as a Netty ChannelInboundHandler, when a corresponding packet is read from the network buffer.
    -   **Outbound (Serialization):** Instances are created using `new BlockBreaking()` by game logic systems (e.g., a WorldManager or PlayerInteraction service) that need to broadcast a change in a block's state. The fields are populated before the object is passed to the network serialization pipeline.

-   **Scope:** BlockBreaking objects are **transient and short-lived**. Their lifecycle is typically confined to the processing of a single network packet or the handling of a single game-tick event. They are not intended to be stored or referenced long-term.

-   **Destruction:** The object becomes eligible for garbage collection immediately after its data has been serialized into a network buffer (outbound) or its contents have been processed by the relevant game system (inbound). No persistent references are maintained.

## Internal State & Concurrency

-   **State:** The object is fully **mutable**. All data fields are public, allowing for direct modification. This design choice prioritizes performance and ease of use within the single-threaded context of the network or game loop, where the object is constructed, populated, and consumed in a linear sequence.

-   **Thread Safety:** This class is **not thread-safe**. It is fundamentally designed for single-threaded access. Concurrent modification from multiple threads without external locking mechanisms will result in a corrupted state and non-deterministic serialization.

    **WARNING:** Never share an instance of BlockBreaking across threads. It must be confined to the thread that creates it, typically a Netty event loop thread or the main game server thread.

## API Surface

The public contract is dominated by static utility methods for serialization and validation, reflecting its role as a self-contained PDU.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | BlockBreaking | O(N) | **[Factory]** Constructs a BlockBreaking instance by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. N is the total length of string data. |
| serialize(buf) | void | O(N) | Encodes the object's state into the provided ByteBuf. This method directly manipulates the buffer's writer index. N is the total length of string data. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a structural integrity check on the binary data in a buffer without full deserialization. Crucial for early rejection of invalid packets. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the current state of the object. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size of a serialized BlockBreaking object already present in a buffer. |

## Integration Patterns

### Standard Usage

A game system creates an instance, populates its fields, and passes it to a network layer for encoding and transmission. Direct calls to `serialize` are rare; this is typically handled by a generic PacketEncoder.

```java
// Executed by a server-side game logic system
BlockBreaking damageState = new BlockBreaking();
damageState.health = 50.0f;
damageState.quantity = 1;
damageState.gatherType = "MINING";
damageState.itemId = "hytale:iron_pickaxe";

// The network layer is responsible for serialization and sending
playerConnection.sendPacket(new BlockDamagePacket(damageState));
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not pool or re-use BlockBreaking instances for different network messages. This is a common source of state leakage bugs. The cost of instantiation is negligible compared to the risk of data corruption.
-   **Multi-threaded Access:** Do not read from or write to a BlockBreaking instance from more than one thread. If data must be passed between threads, create a deep copy or an immutable representation.
-   **Direct Buffer Manipulation:** Avoid calling `serialize` directly unless you are implementing a custom network encoder. The network framework should manage the lifecycle of the ByteBuf.

## Data Pipeline

The BlockBreaking class is a data structure that flows through the network stack. It has two primary pipelines: outbound (serialization) and inbound (deserialization).

**Outbound (e.g., Server to Client):**

> Game Logic System -> **new BlockBreaking(...)** -> Packet Wrapper -> Network Encoder -> `BlockBreaking.serialize(ByteBuf)` -> TCP Socket

**Inbound (e.g., Client from Server):**

> TCP Socket -> Netty ByteBuf -> Protocol Frame Decoder -> **BlockBreaking.deserialize(ByteBuf)** -> Game Event Bus -> Game Logic System

