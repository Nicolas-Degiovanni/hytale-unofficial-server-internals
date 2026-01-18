---
description: Architectural reference for EntityEffect
---

# EntityEffect

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class EntityEffect {
```

## Architecture & Concepts
The EntityEffect class is a data structure, not a service, that represents a temporary status effect applied to an in-game entity. It serves as a fundamental component of the network protocol, acting as the canonical representation for effects like poison, regeneration, or stat modifications when transmitted between the client and server.

Architecturally, its most significant feature is its highly optimized binary serialization format. The format is designed to minimize network bandwidth by dividing the object's data into two distinct sections within the byte buffer:

1.  **Fixed-Size Block:** A 25-byte block at the beginning of the structure containing primitive types and enums. This allows for extremely fast, predictable access to core data like duration and flags.
2.  **Variable-Size Block:** A subsequent block containing data for nullable or variable-length fields such as strings, maps, and nested objects.

A single byte, `nullBits`, at the very start of the serialized data acts as a bitfield. Each bit corresponds to a nullable field, indicating whether its data is present in the variable-size block. This avoids wasting space for null fields. The fixed-size block contains integer offsets that point to the precise location of each variable field's data, enabling efficient parsing and forward-compatibility.

This design pattern is critical for performance in a real-time game engine, prioritizing low-latency access to common fields while accommodating complex, optional data.

## Lifecycle & Ownership
-   **Creation:** An EntityEffect instance is created in one of two ways:
    1.  By the network protocol layer via the static `deserialize` factory method when reading an incoming packet from a Netty ByteBuf.
    2.  By server-side game logic using the `new` keyword to define a new effect that will be applied to an entity and subsequently serialized for network broadcast.
-   **Scope:** Transient. An EntityEffect is a short-lived object. Its lifetime is typically confined to the scope of a single network packet's processing or a single game-tick update. While an entity component may hold an instance for the duration of the effect, the object itself is fundamentally a data carrier, not a persistent state manager.
-   **Destruction:** The object is managed by the Java Garbage Collector and is destroyed when no longer referenced. No manual resource management is required.

## Internal State & Concurrency
-   **State:** Mutable. All public fields are directly accessible and modifiable. The class is designed as a simple data container to be populated, transmitted, and then read.
-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. It is designed to be owned and operated on by a single thread at any given time, such as the main game loop thread or a Netty worker thread.

    **WARNING:** Modifying an EntityEffect instance from multiple threads without explicit, external synchronization will result in data corruption, race conditions, and undefined behavior.

## API Surface
The primary API surface consists of static methods for serialization and validation, reflecting its role as a network DTO.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static EntityEffect | O(N) | Constructs an EntityEffect by reading from a buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the instance's state into the provided buffer according to the defined binary format. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs security and integrity checks on the raw buffer data without full deserialization. Crucial for protecting against malformed packets. |
| computeSize() | int | O(N) | Calculates the total number of bytes the object will occupy when serialized. |
| clone() | EntityEffect | O(N) | Creates a shallow copy of the object, with deep copies for nested collections and objects. |

*N represents the size of the variable-length data fields.*

## Integration Patterns

### Standard Usage
The class is used to transfer state. The server creates and serializes it; the client deserializes it and applies the effect to the local game state.

```java
// Server-side: Creating and preparing an effect for sending
EntityEffect poisonEffect = new EntityEffect();
poisonEffect.id = "hytale:poison";
poisonEffect.name = "Poison";
poisonEffect.duration = 10.0f;
poisonEffect.debuff = true;

// The effect would then be added to a larger packet and serialized.
somePacket.addEffect(poisonEffect);
channel.writeAndFlush(somePacket);

// Client-side: Deserializing from a buffer
// (Assuming 'buf' is a ByteBuf received from the network)
EntityEffect receivedEffect = EntityEffect.deserialize(buf, offset);
playerEntity.applyEffect(receivedEffect);
```

### Anti-Patterns (Do NOT do this)
-   **State Management:** Do not use an EntityEffect instance as a live representation of an active effect on an entity. It is a snapshot for transmission. Game logic should copy its data into a more robust entity-component state machine.
-   **Cross-Thread Sharing:** Never pass an EntityEffect instance between threads for modification. If state must be shared, either pass an immutable copy or serialize it and have the other thread deserialize its own instance.
-   **Skipping Validation:** On a server, never call `deserialize` on a buffer received from a client without first calling `validateStructure`. Bypassing this step exposes the server to denial-of-service attacks via maliciously crafted packets that could cause out-of-memory errors or other crashes.

## Data Pipeline
EntityEffect acts as a payload in the network data flow.

> **Server Flow:**
> Game Event -> Game Logic creates **EntityEffect** -> Serializer writes to ByteBuf -> Network Layer sends Packet

> **Client Flow:**
> Network Layer receives Packet -> Deserializer reads from ByteBuf -> **EntityEffect** is created -> Game Logic applies effect to Entity -> Renderer updates visuals

