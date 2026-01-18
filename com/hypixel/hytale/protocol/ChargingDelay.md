---
description: Architectural reference for ChargingDelay
---

# ChargingDelay

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ChargingDelay {
```

## Architecture & Concepts
The ChargingDelay class is a high-performance Data Transfer Object (DTO) designed for the Hytale network protocol layer. Its sole purpose is to represent a fixed-size, 20-byte data structure that defines the parameters for a "charging" game mechanic, such as drawing a bow or preparing a spell.

This class is not a service or a manager; it is a pure data container. Its design prioritizes raw serialization and deserialization speed over encapsulation. By defining a rigid, fixed-size memory layout (5 little-endian floats), it allows the network layer to read or write the entire data block in a single, highly optimized operation without the overhead of variable-length encoding or field-by-field processing.

It serves as a fundamental building block, often embedded within larger, more complex network packets to describe a component of game state or behavior.

## Lifecycle & Ownership
- **Creation:** An instance of ChargingDelay is created under two distinct circumstances:
    1.  **Inbound:** By the network protocol layer via the static `deserialize` factory method when parsing an incoming network packet from a Netty ByteBuf.
    2.  **Outbound:** By higher-level game logic systems when constructing a new network packet to be sent. The logic populates the fields before passing the object to a packet serializer.

- **Scope:** The object's lifetime is exceptionally short and is strictly scoped to the processing of a single network packet or a single game tick. It is created, its data is used, and it is then immediately discarded, becoming eligible for garbage collection.

- **Destruction:** Destruction is handled automatically by the Java Garbage Collector. There are no native resources or explicit cleanup methods required. Its transient nature ensures a minimal memory footprint.

## Internal State & Concurrency
- **State:** The state of a ChargingDelay object is entirely **mutable**. All five of its data fields are public, allowing for direct, unchecked modification. This design is a deliberate trade-off, sacrificing safety for performance in the tightly controlled environment of the network protocol stack.

- **Thread Safety:** This class is **not thread-safe** and contains no internal synchronization mechanisms.

    **WARNING:** It is critically important to confine usage of a ChargingDelay instance to a single thread. It is typically created and processed on a Netty I/O worker thread or the main game logic thread. Sharing an instance across threads without external locking or creating a defensive copy will result in race conditions, data corruption, and severe application instability.

## API Surface
The public API is minimal and focused exclusively on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static ChargingDelay | O(1) | Factory method. Reads 20 bytes from a buffer at an offset and constructs a new object. |
| serialize(ByteBuf) | void | O(1) | Writes the object's 20 bytes of state into the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the data structure, which is always 20. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-flight check to ensure a buffer contains enough readable bytes for a successful deserialization. |
| clone() | ChargingDelay | O(1) | Creates a new instance with a copy of the current object's field values. |

## Integration Patterns

### Standard Usage
The class is intended to be used as a transient data container during packet processing.

**Deserialization (Receiving Data)**
```java
// Within a network packet decoder, operating on a ByteBuf
if (ChargingDelay.validateStructure(buffer, offset).isOk()) {
    ChargingDelay chargeData = ChargingDelay.deserialize(buffer, offset);
    // Pass the newly created chargeData object to the game logic thread for processing.
    gameLogic.handleChargeParameters(chargeData);
}
```

**Serialization (Sending Data)**
```java
// Within game logic, preparing an outgoing packet
ChargingDelay chargeParams = new ChargingDelay(0.2f, 1.5f, 3.0f, 50.0f, 100.0f);

// The packet builder will subsequently call serialize on the object.
packet.setChargeData(chargeParams);
packet.serialize(outgoingBuffer); // This internally calls chargeParams.serialize()
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not retain and modify a ChargingDelay instance across multiple packets or game ticks. This practice is extremely hazardous due to its mutable state. **Always create a new instance** for each distinct outbound message.
- **Cross-Thread Sharing:** Never pass a ChargingDelay instance from a network thread to a game logic thread without creating a defensive copy via the copy constructor or the `clone` method. Direct sharing is a guaranteed source of concurrency bugs.
- **Direct Instantiation for Deserialization:** Do not use `new ChargingDelay()` and then manually read from a buffer. Always use the static `deserialize` method, which is optimized for this purpose.

## Data Pipeline
ChargingDelay acts as a data record that is hydrated from or dehydrated to a raw byte stream.

> **Inbound Flow:**
> Network ByteBuf -> Packet Decoder -> `ChargingDelay.deserialize` -> **ChargingDelay Instance** -> Game Logic System

> **Outbound Flow:**
> Game Logic System -> **new ChargingDelay(...)** -> Packet Encoder -> `chargingDelay.serialize` -> Network ByteBuf

