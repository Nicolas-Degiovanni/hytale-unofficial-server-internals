---
description: Architectural reference for FogOptions
---

# FogOptions

**Package:** com.hypixel.hytale.protocol
**Type:** Data Structure

## Definition
```java
// Signature
public class FogOptions {
```

## Architecture & Concepts
The FogOptions class is a specialized Data Transfer Object (DTO) that represents the state of atmospheric fog within the game world. It serves as a fundamental component of the Hytale network protocol, acting as a direct, low-level contract between the server and client for synchronizing rendering parameters.

Its design is heavily optimized for network performance. The class maps directly to a **fixed-size 18-byte binary structure**. This fixed-size allocation is a critical architectural choice, eliminating the overhead and complexity of variable-length encoding for this data. This ensures predictable and fast serialization and deserialization, which is essential for real-time game state updates.

FogOptions is not a service or manager; it is a passive data container. Its sole responsibility is to hold fog-related values and provide the logic to convert itself to and from a network-ready byte stream using the Netty ByteBuf.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary scenarios:
    1.  **Client-Side (Deserialization):** The static factory method `deserialize` is called by the network protocol layer when an incoming packet containing fog data is being parsed.
    2.  **Server-Side (Serialization):** The game logic on the server instantiates and populates a FogOptions object to be included in an outgoing packet.

- **Scope:** FogOptions objects are ephemeral and have a very short lifecycle. An instance typically exists only for the duration of processing a single network packet or for applying a state update within a single frame. They are not designed to be long-lived or session-scoped.

- **Destruction:** The object becomes eligible for garbage collection as soon as it has been serialized into a network buffer or its data has been consumed by the client's rendering engine. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The class is entirely mutable. All of its fields are public, allowing for direct modification. It is a plain data structure with no encapsulation, intended for high-performance access within a controlled context. It does not cache any data; it *is* the data.

- **Thread Safety:** **This class is not thread-safe.** Its public mutable fields make it inherently unsafe for concurrent access.

    **WARNING:** FogOptions is designed for use within a single-threaded context, such as a Netty event loop or the main game thread. If an instance must be passed between threads, the sender must create a defensive copy using the `clone()` method or the copy constructor to prevent race conditions. Do not share a single instance across threads without external synchronization.

## API Surface
The public API is dominated by serialization and deserialization logic, reflecting its role in the network protocol.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static FogOptions | O(1) | **Primary Factory.** Constructs a FogOptions instance by reading 18 bytes from a network buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the instance's state as 18 bytes into the provided network buffer. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-flight check to ensure a buffer has enough readable bytes for a successful deserialization. |
| clone() | FogOptions | O(1) | Creates a value-identical copy of the instance. Essential for thread-safe operations. |

## Integration Patterns

### Standard Usage
The intended use involves the server creating an instance, populating it with current game state, and serializing it into a larger packet. The client receives this packet, deserializes the FogOptions data, and passes it to the rendering system.

```java
// Server-side: Preparing a packet
FogOptions fog = new FogOptions();
fog.fogFarViewDistance = 300.0f;
fog.ignoreFogLimits = false;
// ... set other fields

somePacket.setFogOptions(fog);
networkManager.send(somePacket);

// Client-side: Handling a packet
// The protocol decoder calls this internally
FogOptions receivedFog = FogOptions.deserialize(packetBuffer, offset);
renderingEngine.applyFog(receivedFog);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not modify and reuse the same FogOptions instance over multiple game ticks or for multiple outgoing packets. This is highly error-prone due to its mutable nature. Always create a new, clean instance for each distinct state update.
- **Concurrent Modification:** Never modify a FogOptions instance from one thread while another thread is reading its fields or calling its `serialize` method. This will lead to data corruption.
- **Partial Deserialization:** Do not attempt to read the fields from the ByteBuf manually. Always rely on the `deserialize` method to ensure correct byte order (Little Endian) and field mapping.

## Data Pipeline
The FogOptions class is a critical link in the state synchronization pipeline between the server and the client's renderer.

> **Server Flow:**
> Game World State -> **FogOptions (Instance)** -> `serialize()` -> Outgoing Packet (ByteBuf) -> Network Layer

> **Client Flow:**
> Network Layer -> Incoming Packet (ByteBuf) -> `deserialize()` -> **FogOptions (Instance)** -> Rendering Engine

