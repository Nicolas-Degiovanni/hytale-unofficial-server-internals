---
description: Architectural reference for WorldParticle
---

# WorldParticle

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class WorldParticle {
```

## Architecture & Concepts
The WorldParticle class is a data-centric object that serves as a direct, low-level representation of a particle effect within the Hytale network protocol. It is not a manager or a service; it is a fundamental data structure, analogous to a message in other serialization frameworks.

Its primary architectural role is to act as a standardized container for the properties of a single visual particle effect that needs to be communicated between the client and server. The class design is heavily optimized for performance, specifically for high-frequency binary serialization and deserialization.

The binary layout is a key aspect of its design. It employs a fixed-plus-variable block structure:
1.  **Fixed Block (32 bytes):** Contains a one-byte nullability bitmask followed by fixed-size fields like scale, color, position, and rotation. This allows for extremely fast, predictable reads of the most common data.
2.  **Variable Block:** Follows the fixed block and contains variable-length data, such as the string-based systemId.

This structure minimizes network overhead and CPU cost during packet processing, which is critical for features like particle systems that can generate a high volume of network traffic.

## Lifecycle & Ownership
-   **Creation:** WorldParticle instances are created under two primary circumstances:
    1.  **Inbound:** The network protocol layer instantiates it via the static deserialize method when parsing an incoming packet from a Netty ByteBuf.
    2.  **Outbound:** Game logic systems (e.g., spell effects, block breaking logic, animation controllers) instantiate it using its constructor when preparing to send a packet that spawns a particle.

-   **Scope:** The lifetime of a WorldParticle object is exceptionally short and transient. It is designed to exist only for the duration of a single transaction—either the construction of an outbound packet or the processing of an inbound one. Once its data has been serialized to a buffer or consumed by a game system, it is immediately eligible for garbage collection.

-   **Destruction:** Cleanup is handled exclusively by the Java Garbage Collector. There are no native resources or manual memory management responsibilities associated with this class.

## Internal State & Concurrency
-   **State:** The object is fully mutable, with all fields exposed publicly. This is a deliberate performance-oriented design choice to eliminate the overhead of getter and setter method calls during the critical path of packet serialization and deserialization. It is a pure data holder and does not cache any information.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access within the context of a network I/O worker or a game tick. Sharing an instance across threads without explicit, external synchronization will result in data corruption and undefined behavior. Any handoff between threads, such as from a network thread to the main game loop, must be done by value (creating a new instance) or with appropriate locking.

## API Surface
The public API is focused entirely on serialization, deserialization, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static WorldParticle | O(N) | Constructs a new WorldParticle by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. Essential for pre-allocating buffers. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a lightweight check on a buffer to verify if it contains a structurally valid WorldParticle without full deserialization. |
| clone() | WorldParticle | O(N) | Creates a deep copy of the object. Useful for safely passing the object's state across thread boundaries. |

*Complexity O(N) refers to the length of the variable-sized systemId string.*

## Integration Patterns

### Standard Usage
A WorldParticle is typically created by game logic, configured, and then passed to a higher-level packet object for transmission.

```java
// Example: A game system spawning a custom particle effect at a location.
WorldParticle magicSpark = new WorldParticle();
magicSpark.systemId = "hytale:spell.arcane_burst";
magicSpark.scale = 2.5f;
magicSpark.color = new Color(255, 100, 255);
magicSpark.positionOffset = new Vector3f(0.0f, 1.0f, 0.0f);

// The configured object is then added to a packet payload.
// The network layer will later call magicSpark.serialize() internally.
SpawnEffectPacket packet = new SpawnEffectPacket();
packet.addParticle(magicSpark);
networkManager.sendToClients(packet);
```

### Anti-Patterns (Do NOT do this)
-   **Object Re-use:** Do not retain and modify a single WorldParticle instance across multiple packets or game ticks. The performance gain is negligible and it creates a high risk of state corruption. Always create a new instance for each distinct effect.
-   **Cross-Thread Sharing:** Never pass a WorldParticle instance from a network thread to the main game thread directly. The receiving thread must create a defensive copy using the clone method to ensure thread safety.
-   **Direct Instantiation for Deserialization:** Do not use `new WorldParticle()` and then attempt to populate its fields from a buffer manually. Always use the static `WorldParticle.deserialize(buffer, offset)` method, which correctly handles the complex binary layout, including the nullability bitmask and variable-length fields.

## Data Pipeline
WorldParticle is not a processing stage but rather the data payload itself. It flows through the system as follows.

**Outbound (Server to Client):**
> Flow:
> Game Logic System -> **WorldParticle Instance** -> Packet Object -> Packet Serializer -> Netty ByteBuf -> Network Socket

**Inbound (Client to Server):**
> Flow:
> Network Socket -> Netty ByteBuf -> Packet Deserializer -> **WorldParticle Instance** -> Game Event Bus -> Particle Rendering System

