---
description: Architectural reference for SleepMultiplayer
---

# SleepMultiplayer

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Transient Data Object

## Definition
```java
// Signature
public class SleepMultiplayer {
```

## Architecture & Concepts
The SleepMultiplayer class is a Data Transfer Object (DTO) that represents a network packet. Its sole purpose is to encapsulate and transport the state of the multiplayer sleeping mechanic between the server and connected clients. It is a fundamental component of the game's state synchronization protocol.

This class does not contain any game logic. It acts as a structured data container that is created by the network layer during deserialization or by the game logic layer before serialization. Its design is heavily optimized for performance and network efficiency, utilizing low-level byte manipulation with a Netty ByteBuf. The serialization format is fixed and includes a bitmask for handling nullable fields, a fixed-size data block, and a variable-length array of UUIDs.

This object is a low-level implementation detail of the network protocol and should not be directly referenced by high-level game systems. Instead, its data should be extracted and passed to services responsible for managing game state.

## Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1. **Inbound:** The static factory method `deserialize` is invoked by a network pipeline handler (e.g., a Netty ChannelInboundHandler) when a corresponding packet ID is read from the network stream.
    2. **Outbound:** The game server's logic (e.g., a World or Player management system) instantiates it directly using its constructor when it needs to broadcast an update about sleeping players.
- **Scope:** The object is extremely short-lived and transient. It is designed to exist only within the scope of a single network event processing cycle.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as the network handler or game logic method that created it completes its execution. No manual cleanup is required. **WARNING:** Holding long-lived references to this object is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** The class is a mutable container for sleep-related data. All its fields are public and can be modified after instantiation. The internal state consists of primitive integers and a potentially nullable array of UUIDs.
- **Thread Safety:** **CRITICAL:** This class is **not thread-safe**. It is designed for use within a single-threaded context, such as a Netty event loop. All serialization, deserialization, and data access must occur on the same thread. Unsynchronized access from multiple threads will lead to race conditions, data corruption, and unpredictable behavior. Any handoff of its data to other threads must be done by copying the data into a thread-safe structure, not by passing a reference to the instance itself.

## API Surface
The public API is focused entirely on protocol-level operations for encoding and decoding the packet data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | SleepMultiplayer | O(N) | Static factory. Reads from a ByteBuf to construct a new instance. N is the number of UUIDs in the awakeSample. Throws ProtocolException on data corruption or bounds violations. |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided ByteBuf according to the protocol specification. Throws ProtocolException if array limits are exceeded. |
| computeSize() | int | O(1) | Calculates the exact number of bytes this object will occupy on the network when serialized. Does not perform the actual serialization. |
| computeBytesConsumed(ByteBuf, int) | int | O(1) | Calculates the size of an encoded packet directly from a buffer without full deserialization. Used for skipping packets or validation. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a lightweight check on a buffer to ensure it contains a structurally valid packet. Protects against malformed data and potential exploits. |

## Integration Patterns

### Standard Usage
The primary integration pattern involves a network handler decoding the packet and passing its data to a higher-level system, often via an event bus or a scheduled task queue to avoid blocking the network thread.

```java
// In a Netty ChannelInboundHandler or similar context
// 1. A ByteBuf is received from the network
ByteBuf networkBuffer = ...;

// 2. The packet is deserialized into a transient object
SleepMultiplayer packet = SleepMultiplayer.deserialize(networkBuffer, 0);

// 3. The data is extracted and passed to the main game loop or a service
//    Do NOT pass the packet object itself.
int sleepers = packet.sleepersCount;
int awake = packet.awakeCount;
UUID[] sample = packet.awakeSample;

gameContext.getSleepManager().onStateUpdate(sleepers, awake, sample);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Retention:** Do not store instances of SleepMultiplayer in services or components as part of the game state. They are transient messages, not persistent entities.
- **Cross-Thread Modification:** Do not create a packet on one thread and serialize it on another without explicit and correct synchronization. The object is not designed for this and will fail unpredictably.
- **Object Pooling:** Do not attempt to pool and reuse these objects. The overhead of managing a pool outweighs the negligible cost of garbage collecting these small, short-lived instances. Incorrectly resetting a pooled object can lead to severe data corruption bugs.

## Data Pipeline
The SleepMultiplayer class is a critical link in the flow of game state data between the network and the game simulation.

> **Inbound Flow (Receiving Data):**
> Netty ByteBuf -> **SleepMultiplayer.deserialize** -> Transient SleepMultiplayer Instance -> Game Logic (Data Extraction) -> Game State Update

> **Outbound Flow (Sending Data):**
> Game Event (e.g., Player Sleeps) -> Game Logic (Creates Instance) -> **SleepMultiplayer.serialize** -> Netty ByteBuf -> Network Socket

