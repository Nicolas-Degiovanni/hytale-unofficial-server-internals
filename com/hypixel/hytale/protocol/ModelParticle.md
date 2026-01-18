---
description: Architectural reference for ModelParticle
---

# ModelParticle

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Model

## Definition
```java
// Signature
public class ModelParticle {
```

## Architecture & Concepts
The ModelParticle class is a data transfer object (DTO) that serves as a fundamental component of the Hytale network protocol. It is not an active game-world object but rather a serialized data contract that describes how to render a particle effect attached to a 3D model. Its primary function is to facilitate the efficient transmission of particle effect parameters between the client and server.

The core architectural principle of this class is its highly optimized, custom binary serialization format. This format is designed to minimize network bandwidth and reduce CPU overhead during parsing. It achieves this through several key techniques:

1.  **Nullable Bit Field:** The first byte of the serialized data is a bitmask that declares which of the object's nullable fields are present. This avoids the need to transmit null markers for every optional field, saving significant space.
2.  **Fixed and Variable Data Blocks:** The serialized structure is split into a fixed-size block for predictable data types (floats, booleans, enums) and a variable-size block for data of unknown length (strings).
3.  **Offset-Based Pointers:** The fixed-size block contains integer offsets that point to the location of corresponding data within the variable-size block. This allows for extremely fast, non-sequential parsing and validation, as the parser can jump directly to the data it needs without reading everything in between.

This class is a pure data container; it holds no logic related to rendering or game state updates. It is the raw blueprint that other systems, such as the Visual Effect Manager or the rendering engine, use to instantiate and manage actual particle effects in the game world.

## Lifecycle & Ownership
-   **Creation:** A ModelParticle instance is created in one of two scenarios:
    1.  **Deserialization:** The network layer instantiates it by calling the static `deserialize` method when processing an incoming network packet. This is the most common creation path on the receiving end (e.g., a client receiving effect data from the server).
    2.  **Direct Instantiation:** Game logic instantiates it using one of its constructors when preparing to send particle effect data. For example, a weapon system might create a ModelParticle to describe a muzzle flash effect.

-   **Scope:** The object is **transient** and **short-lived**. Its scope is typically confined to a single network packet processing cycle or a single frame's logic update. It is not managed by a central registry and should not be stored long-term.

-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection as soon as it is no longer referenced, which is usually after its data has been consumed by a downstream system (e.g., the rendering engine). There are no manual cleanup or destruction methods.

## Internal State & Concurrency
-   **State:** The state of a ModelParticle is entirely **mutable**. All of its public fields can be directly accessed and modified after creation. It acts as a simple, open data structure. It performs no internal caching.

-   **Thread Safety:** This class is **not thread-safe**. It is designed for use within a single-threaded context, such as a Netty I/O thread or the main game loop. Accessing or modifying a ModelParticle instance from multiple threads without external synchronization will result in undefined behavior and data corruption.

## API Surface
The public API is dominated by static methods for serialization, deserialization, and validation, reflecting its role as a protocol-level data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ModelParticle | O(N) | Constructs a ModelParticle by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the custom binary format. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a security-critical check on a ByteBuf to ensure the data is well-formed *before* attempting deserialization. |
| computeBytesConsumed(buf, offset) | static int | O(V) | Calculates the total number of bytes the object occupies in a buffer, including variable-length fields. V is the number of variable fields. |
| computeSize() | int | O(V) | Calculates the total size the object will require when serialized. |

*N = Total size of variable-length data (strings). V = Number of variable-length fields.*

## Integration Patterns

### Standard Usage
A developer will almost never interact with this class directly unless implementing low-level networking or game logic systems. The typical flow involves either creating an instance to be sent or receiving one that has been deserialized.

**Sending a Particle Effect:**
```java
// Game logic creates a descriptor for a "fire" particle on a model's hand
ModelParticle fireEffect = new ModelParticle(
    "hytale:fire_small",
    1.0f,
    null, // No color tint
    EntityPart.Model,
    "r_hand_attach", // Attach to a specific model node
    new Vector3f(0, 0.1f, 0),
    null,
    false
);

// The packet building system will then call serialize
// outgoingPacket.writeModelParticle(fireEffect);
// which internally calls:
// fireEffect.serialize(byteBuffer);
```

**Receiving and Processing a Particle Effect:**
```java
// In a network packet handler...
// First, validate the incoming data to prevent exploits
ValidationResult result = ModelParticle.validateStructure(buf, offset);
if (!result.isOk()) {
    throw new ProtocolException("Invalid ModelParticle: " + result.getReason());
}

// If valid, deserialize the object
ModelParticle receivedEffect = ModelParticle.deserialize(buf, offset);

// Pass the data to the appropriate game system
visualEffectManager.spawnParticleFromDescriptor(entityId, receivedEffect);
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not cache and reuse ModelParticle instances across multiple ticks or for different effects. They are lightweight DTOs and should be created on-demand to avoid complex state management issues.
-   **Ignoring Validation:** On a server, failing to call `validateStructure` on untrusted client data before calling `deserialize` is a critical security vulnerability. A malicious client could send a packet with invalid offsets or lengths, leading to server exceptions or crashes.
-   **Cross-Thread Access:** Do not create a ModelParticle on the main thread and pass its reference to a worker thread for modification without proper synchronization. This will lead to race conditions.

## Data Pipeline
The ModelParticle class is a data payload that flows through the network and game engine.

**Inbound Flow (e.g., Server to Client):**
> Network ByteBuf -> Protocol Framer -> **ModelParticle.validateStructure** -> **ModelParticle.deserialize** -> ModelParticle Instance -> Visual Effect System -> Particle Renderer

**Outbound Flow (e.g., Client to Server):**
> Game Logic Event -> **new ModelParticle()** -> ModelParticle Instance -> Packet Building System -> **ModelParticle.serialize** -> Network ByteBuf -> Protocol Framer

