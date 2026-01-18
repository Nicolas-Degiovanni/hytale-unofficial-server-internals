---
description: Architectural reference for Rangeb
---

# Rangeb

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class Rangeb {
```

## Architecture & Concepts
The Rangeb class is a low-level, fixed-size data structure designed for network serialization. It represents a fundamental component of the Hytale network protocol, encapsulating a simple range defined by a minimum and maximum byte value.

This class is not a service or a manager; it is a passive data container, analogous to a C-style struct. Its primary responsibility is to provide a standardized, high-performance mechanism for mapping between an in-memory object representation and its on-the-wire byte format. The design prioritizes serialization speed and a predictable memory layout, evident from the static constants like FIXED_BLOCK_SIZE which define its exact 2-byte wire size.

Rangeb objects are intended to be composed within larger, more complex protocol message classes. They serve as building blocks for defining data fields that require a bounded byte range, such as item durability, entity health ranges, or procedural generation parameters.

### Lifecycle & Ownership
-   **Creation:** Instances are created on-demand during two primary scenarios:
    1.  **Deserialization:** The static deserialize method constructs a new Rangeb when parsing an incoming network packet from a Netty ByteBuf.
    2.  **Serialization:** A higher-level system constructs a new Rangeb to populate an outgoing network packet before calling the serialize method.
-   **Scope:** Ephemeral. The lifetime of a Rangeb instance is extremely short, typically confined to the scope of a single network message handler or packet construction method. It is not intended for long-term storage.
-   **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as all references to it are dropped, which usually occurs immediately after the parent network packet has been processed or transmitted.

## Internal State & Concurrency
-   **State:** Mutable. The public fields *min* and *max* can be directly accessed and modified after instantiation. This design choice favors performance over encapsulation, a common trade-off in low-level networking code.
-   **Thread Safety:** **Not thread-safe.** This class contains no internal synchronization mechanisms. Instances must be confined to the thread that created them, which is almost always a Netty I/O worker thread. Sharing a Rangeb instance across threads without explicit external locking will lead to race conditions and unpredictable behavior.

## API Surface
The public contract is focused entirely on serialization, deserialization, and size calculation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static Rangeb | O(1) | Constructs a new Rangeb by reading 2 bytes from the provided ByteBuf at a specific offset. |
| serialize(buf) | void | O(1) | Writes the min and max fields as two consecutive bytes into the provided ByteBuf. |
| computeSize() | int | O(1) | Returns the fixed size of the object on the wire, which is always 2. |
| computeBytesConsumed(buf, offset) | static int | O(1) | Returns the fixed number of bytes this object occupies in a buffer, which is always 2. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a bounds check to ensure the buffer contains enough readable bytes for a valid Rangeb. |

## Integration Patterns

### Standard Usage
The class is used as part of a larger packet serialization or deserialization process. It is never used in isolation for business logic.

```java
// Example: Writing and reading a Rangeb
// Assumes a pre-allocated Netty ByteBuf

// --- Serialization ---
Rangeb damageRange = new Rangeb((byte) 10, (byte) 25);
damageRange.serialize(byteBuf);

// --- Deserialization ---
// In a network handler, after validating buffer size
Rangeb receivedRange = Rangeb.deserialize(byteBuf, 0);
byte minDamage = receivedRange.min;
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term State:** Do not use Rangeb to store application state. It is a transient DTO for network communication only. Its mutable nature makes it unsuitable for representing stable state.
-   **Cross-Thread Sharing:** Never pass a Rangeb instance from a network thread to a game logic thread without first copying its values into a different, thread-safe data structure.
-   **Manual Memory Management:** Do not attempt to manage the lifecycle of this object. Rely on standard Java garbage collection.

## Data Pipeline
Rangeb sits at one of the lowest levels of the data serialization pipeline, acting as a direct interface to the raw byte stream.

> **Inbound Flow:**
> Netty ByteBuf -> **Rangeb.deserialize** -> Rangeb Instance -> Parent Protocol Message -> Game Event Handler

> **Outbound Flow:**
> Game Logic -> Parent Protocol Message -> Rangeb Instance -> **Rangeb.serialize** -> Netty ByteBuf

