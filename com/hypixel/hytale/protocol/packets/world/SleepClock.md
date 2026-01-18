---
description: Architectural reference for SleepClock
---

# SleepClock

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class SleepClock {
```

## Architecture & Concepts
The SleepClock class is a struct-like Data Transfer Object, not a service or manager. It exists purely to encapsulate and transport the state of the in-game sleeping mechanic over the network. This mechanic involves an accelerated passage of time, and SleepClock provides the necessary data for clients to synchronize their visual representation of this process with the server's authoritative state.

As a component of the Hytale network protocol, it is designed for extreme performance and low garbage collection overhead. Its design favors direct field access over getter/setter methods to eliminate method call overhead in the performance-critical network deserialization path. The class is fundamentally a data contract for a fixed-block segment of a larger network packet.

Key architectural characteristics include:
- **Fixed-Size Layout:** The object serializes to a predictable, fixed block of 33 bytes, simplifying buffer management and parsing logic.
- **Null-Value Optimization:** A bitmask is used as the first byte of the serialized data to efficiently indicate whether the nullable InstantData fields are present, avoiding the need for larger markers or conditional logic that depends on stream position.
- **Network-Centric API:** The primary interface is through its static `deserialize` and instance `serialize` methods, which operate directly on Netty ByteBuf instances.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1.  **On the Server:** The game logic instantiates and populates a SleepClock with the current world state before serializing it into an outgoing packet.
    2.  **On the Client:** The static `deserialize` factory method instantiates a SleepClock when parsing an incoming network packet from the server.

- **Scope:** The lifecycle of a SleepClock instance is exceptionally brief and transient. It is scoped to the processing of a single network packet. Once its data has been transferred to or from the canonical game state, the object is immediately dereferenced and becomes eligible for garbage collection.

- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual resource management or `close` methods. Holding long-term references to SleepClock objects is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** The object's state is fully mutable via its public fields. It is a simple container for data and contains no internal logic, caching, or derived state.

- **Thread Safety:** **WARNING:** This class is not thread-safe and must not be shared across threads. It is designed to be created, populated, and read by a single thread, typically a Netty I/O worker thread. Any concurrent access requires external synchronization, which would violate the intended high-performance usage pattern of this class.

## API Surface
The public contract is focused on serialization, deserialization, and validation for the network layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static SleepClock | O(1) | Constructs a new SleepClock by reading a fixed 33-byte block from the buffer at the given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state into a fixed 33-byte block in the provided buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough readable bytes for a valid object. |
| clone() | SleepClock | O(1) | Creates a deep copy of the object, including its contained InstantData fields. |

## Integration Patterns

### Standard Usage
SleepClock is never used in isolation. It is always deserialized as part of a larger packet-handling routine and its data is immediately transferred to a more permanent game state manager.

```java
// Example from a hypothetical client-side packet handler
public void handleWorldStatePacket(ByteBuf packetBuffer) {
    // ... read other packet data up to the SleepClock offset ...
    int currentOffset = 64; // Example offset

    // Validate and deserialize the SleepClock data
    if (SleepClock.validateStructure(packetBuffer, currentOffset).isError()) {
        throw new DeserializationException("Invalid SleepClock data in packet");
    }
    SleepClock clockData = SleepClock.deserialize(packetBuffer, currentOffset);

    // Immediately apply the transient data to the persistent game systems
    WorldTimeManager timeManager = this.game.getWorldTimeManager();
    timeManager.updateSleepState(
        clockData.startGametime,
        clockData.targetGametime,
        clockData.progress
    );

    // The clockData object is now out of scope and will be garbage collected.
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store SleepClock instances in fields or collections. They represent a point-in-time snapshot. Copy their values into your own state objects and discard the DTO.
- **Cross-Thread Sharing:** Do not pass a SleepClock instance from the network thread to a game logic or rendering thread. This will introduce race conditions. Instead, pass the primitive values or immutable copies.
- **Manual Serialization:** Do not attempt to read from a ByteBuf and populate a `new SleepClock()` manually. The static `deserialize` method correctly handles the null-bitmask and byte ordering, and bypassing it will lead to data corruption.

## Data Pipeline
The SleepClock acts as a data vessel in the server-to-client world state synchronization pipeline.

> **Server Flow:**
> Authoritative Game State (WorldTimeManager) -> **SleepClock** instance created -> `serialize()` method called -> Raw bytes written to network ByteBuf -> Sent to Client

> **Client Flow:**
> Raw bytes received in network ByteBuf -> `deserialize()` method called -> **SleepClock** instance created -> Data copied to client-side Game State (WorldTimeManager) -> UI/Rendering Update

