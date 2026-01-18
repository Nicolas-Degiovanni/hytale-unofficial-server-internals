---
description: Architectural reference for HorizontalSelector
---

# HorizontalSelector

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class HorizontalSelector extends Selector {
```

## Architecture & Concepts
The HorizontalSelector is a low-level network protocol data structure, not a service or manager. Its primary function is to define and serialize a fixed-size, 34-byte geometric volume used for entity selection and targeting within the game world. It represents a concrete implementation of the abstract Selector pattern.

This class is designed for high-performance network I/O and sits at the boundary between the game logic and the raw network layer. Its rigid, C-style structure with a fixed block size of 34 bytes eliminates the overhead associated with variable-length data encoding, making serialization and deserialization deterministic and extremely fast.

The fields, such as yawLength, pitchOffset, and startDistance, mathematically describe a selection frustum or cone originating from an entity. This data is used by the server-side simulation to perform spatial queries, such as determining which entities are affected by an attack, visible to an AI, or targeted by a player ability.

## Lifecycle & Ownership
- **Creation:** A HorizontalSelector is instantiated under two conditions:
    1.  **Inbound:** The static `deserialize` factory method is called by the network protocol decoder when a corresponding packet is read from a Netty ByteBuf.
    2.  **Outbound:** Game logic, such as a combat or AI system, instantiates it directly via its constructor to define a targeting volume before serializing it into an outgoing packet.
- **Scope:** The object's lifetime is ephemeral. It is scoped to the processing of a single network packet or a single game-tick operation. It is not designed to be cached or persist across frames.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as the reference to it is dropped, typically after the parent network packet has been fully processed or the relevant game logic has completed.

## Internal State & Concurrency
- **State:** The state is fully **Mutable**. All fields are public and can be directly modified after creation. This class acts as a pure data container with no internal logic, caching, or complex state management.
- **Thread Safety:** This class is **not thread-safe**. It contains no locks, volatile keywords, or any other concurrency primitives.

    **WARNING:** Accessing a HorizontalSelector instance from multiple threads without external synchronization is unsafe. For example, modifying its fields on the main game thread while a Netty I/O thread is serializing it will result in packet corruption or unpredictable behavior. Confine usage to a single thread or implement an explicit locking strategy around it.

## API Surface
The public API is focused exclusively on serialization, deserialization, and validation for network transport.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static HorizontalSelector | O(1) | Constructs a new HorizontalSelector by reading 34 bytes from a buffer at a given offset. |
| serialize(ByteBuf) | int | O(1) | Writes the object's 34 bytes of state into the provided buffer. Returns the number of bytes written. |
| computeSize() | int | O(1) | Returns the constant size of the serialized structure, which is always 34. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-flight check to ensure a buffer contains enough readable bytes to deserialize the object. |
| clone() | HorizontalSelector | O(1) | Creates a new instance with an identical copy of the data. |

## Integration Patterns

### Standard Usage
The class is intended for direct use within packet serialization and deserialization routines.

```java
// Outbound: Creating and sending a selector
HorizontalSelector selector = new HorizontalSelector(
    1.0f, 1.0f, 90.0f, 0.0f, 0.0f, 0.0f, 0.5f, 10.0f, 
    HorizontalSelectorDirection.ToLeft, true
);

ByteBuf buffer = getPacketBuffer();
selector.serialize(buffer);
sendPacket(buffer);

// Inbound: Reading and using a selector
ByteBuf receivedBuffer = ...;
int offset = ...; // Start of the selector data in the buffer
ValidationResult result = HorizontalSelector.validateStructure(receivedBuffer, offset);

if (result.isOk()) {
    HorizontalSelector receivedSelector = HorizontalSelector.deserialize(receivedBuffer, offset);
    // Use the selector in game logic...
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify and reuse the same HorizontalSelector instance for multiple distinct network packets. This practice is highly error-prone due to its mutable nature. Always create a new instance or use `clone()` for each logical operation.
- **Cross-Thread Sharing:** Do not pass a HorizontalSelector instance from a game logic thread to a network I/O thread (or vice-versa) without a clear synchronization contract. This is a direct path to race conditions.

## Data Pipeline
As a data structure, HorizontalSelector does not process data itself; it *is* the data being processed. Its flow through the engine is linear and bidirectional.

**Outbound Flow (Client or Server -> Network):**
> Game Logic System -> **HorizontalSelector (Instance Created)** -> `serialize()` -> Netty ByteBuf -> Network Socket

**Inbound Flow (Network -> Client or Server):**
> Network Socket -> Netty ByteBuf -> `deserialize()` -> **HorizontalSelector (Instance Created)** -> Packet Handler / Game Logic System

