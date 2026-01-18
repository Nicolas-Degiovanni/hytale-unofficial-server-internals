---
description: Architectural reference for SavedMovementStates
---

# SavedMovementStates

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class SavedMovementStates {
```

## Architecture & Concepts
The SavedMovementStates class is a low-level Data Transfer Object (DTO) designed for network serialization. It is not a service or manager, but rather a structured data container representing a snapshot of a player's movement capabilities at a specific moment.

Its primary role is to facilitate the efficient transfer of the `flying` state between the client and server. The class is part of a larger, likely auto-generated, protocol framework. This is evidenced by its rigid structure, static serialization and validation methods, and compile-time constants like `FIXED_BLOCK_SIZE` and `MAX_SIZE`.

In the context of the Hytale engine, this object acts as a payload fragment. It is not a complete network packet itself but is embedded within larger packet structures that handle player state synchronization. Its design prioritizes performance and predictability in a high-throughput networking environment, sacrificing flexibility for a guaranteed memory layout and size.

### Lifecycle & Ownership
- **Creation:** Instances are created under two conditions:
    1. **Inbound:** The static `deserialize` method instantiates the object when the network layer decodes an incoming packet from a Netty ByteBuf.
    2. **Outbound:** Game logic instantiates the object via its constructor (`new SavedMovementStates(isFlying)`) when preparing to send a player state update to the network.
- **Scope:** The lifecycle of a SavedMovementStates instance is extremely brief and tied to the processing of a single network packet. It is a temporary, stack-allocated object in most scenarios.
- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been read by the game logic (inbound) or written to a network buffer (outbound). **WARNING:** Holding references to these objects beyond the scope of a single packet-processing event is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** The object's state is minimal and fully mutable, consisting of a single public boolean field, `flying`. It contains no internal caches or complex data structures.
- **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded access within the context of a Netty I/O worker thread or the main game thread. All read and write operations on its fields must be externally synchronized if concurrent access is unavoidable. Unsynchronized access will lead to data corruption and unpredictable behavior.

## API Surface
The public contract is dominated by static utility methods used by the protocol framework for serialization and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | SavedMovementStates | O(1) | **[Static]** Constructs an object by reading a single byte from the provided buffer at the given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the `flying` state as a single byte (1 or 0) into the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized data, which is always 1 byte. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | **[Static]** Performs a bounds check to ensure the buffer is large enough to contain the object's data. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by high-level game logic developers. It is an implementation detail of the network protocol layer. The typical interaction is managed by the packet serialization/deserialization pipeline.

An example of the protocol layer deserializing this object:
```java
// Within a packet's deserialization logic
// The buffer contains the full packet payload

// First, validate there is enough data to read
ValidationResult result = SavedMovementStates.validateStructure(packetBuffer, offset);
if (!result.isOk()) {
    // Handle corrupted or incomplete packet
    throw new ProtocolException(result.getErrorMessage());
}

// Deserialize the data structure
SavedMovementStates movementState = SavedMovementStates.deserialize(packetBuffer, offset);

// Apply the state to the relevant game entity
player.setFlying(movementState.flying);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not use a SavedMovementStates instance as a component on a player entity or to store game state. It is a transient snapshot, not a persistent state container. Copy its value into the authoritative game state immediately after deserialization.
- **Cross-Thread Sharing:** Do not pass an instance of this class between threads without proper locking. It is fundamentally unsafe for concurrent modification.
- **Ignoring Validation:** Never call `deserialize` without first calling `validateStructure`. Doing so can lead to buffer underflow exceptions and server instability if presented with malformed packets.

## Data Pipeline
The class serves as a simple data marshalling step in the network pipeline.

> **Inbound Flow:**
> Raw TCP Packet -> Netty ByteBuf -> Protocol Packet Decoder -> **SavedMovementStates.deserialize** -> Player Entity State Update

> **Outbound Flow:**
> Player Entity State Change -> Protocol Packet Encoder -> **new SavedMovementStates()** -> **serialize(ByteBuf)** -> Netty ByteBuf -> Raw TCP Packet

---

