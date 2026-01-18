---
description: Architectural reference for FluidFXMovementSettings
---

# FluidFXMovementSettings

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class FluidFXMovementSettings {
```

## Architecture & Concepts
The FluidFXMovementSettings class is a specialized Data Transfer Object (DTO) designed for the Hytale network protocol. It is not a service or manager, but rather a simple, fixed-size data container that represents a specific block of bytes in a network stream.

Its primary responsibility is to define the physical and perceptual properties of a player's interaction with a fluid volume. This includes parameters governing movement speed, sinking behavior, and camera field-of-view adjustments when submerged.

Architecturally, this class serves as both the data model and the codec for its corresponding network data. The static methods, particularly deserialize and validateStructure, are integral components of the protocol decoding pipeline. They operate directly on Netty ByteBuf instances, translating a raw byte stream into a structured, in-memory representation that the game engine can understand and apply. The fixed block size of 24 bytes (6 floats * 4 bytes/float) underscores its role as a low-level, performance-critical component where predictable data layout is paramount.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary scenarios:
    1.  **Deserialization:** The network protocol layer instantiates FluidFXMovementSettings via the static deserialize method when processing an incoming network packet that contains fluid definition data.
    2.  **Configuration:** Game logic or content pipelines may instantiate this class to define the properties of a new fluid type, which is then serialized and transmitted to clients.
- **Scope:** This object is ephemeral and has a very short lifecycle. Its scope is typically confined to the processing of a single network packet or a single game logic update. It is not designed to be a long-lived, persistent entity.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as all references to it are dropped, which typically occurs immediately after its data has been copied into the relevant physics or rendering components of the game engine.

## Internal State & Concurrency
- **State:** The internal state is fully mutable. All fields are public primitives, allowing for direct, unrestricted modification. This design prioritizes performance and ease of access over encapsulation, which is typical for low-level DTOs. The class holds no caches or derived state.
- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization mechanisms. It is designed to be created, populated, and read within a single thread, such as the main game loop or a dedicated network processing thread.

    **WARNING:** Sharing an instance of FluidFXMovementSettings across multiple threads without explicit external synchronization will result in race conditions, memory visibility issues, and unpredictable application behavior.

## API Surface
The public contract is dominated by static methods for protocol handling and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static FluidFXMovementSettings | O(1) | Constructs a new instance by reading 24 bytes from a buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the 6 float fields into the provided buffer using little-endian byte order. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains sufficient bytes for a valid structure. |
| computeSize() | int | O(1) | Returns the fixed size of the serialized data, which is always 24. |

## Integration Patterns

### Standard Usage
This class is not typically used directly by high-level gameplay code. It is an implementation detail of the protocol and world data systems. A protocol handler would use it to decode a segment of a larger packet.

```java
// Hypothetical usage within a protocol decoder
public void handleZoneDataPacket(ByteBuf packet) {
    int offset = ... // Calculate start of the fluid settings
    
    ValidationResult result = FluidFXMovementSettings.validateStructure(packet, offset);
    if (!result.isOk()) {
        // Handle malformed packet
        return;
    }

    FluidFXMovementSettings settings = FluidFXMovementSettings.deserialize(packet, offset);
    
    // Apply the deserialized settings to the game world
    gameWorld.getActiveZone().setFluidPhysics(settings);
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not reuse a single instance of this class to process multiple network packets. This can lead to state bleeding and unpredictable behavior. Always create a new instance per data block.
- **Multi-threaded Access:** Never pass an instance to another thread for modification. If data needs to be shared, create a deep copy using the copy constructor or the clone method first.
- **Manual Serialization:** Avoid manually writing the float values to a buffer. Always use the provided serialize method to ensure the correct byte order (little-endian) and field sequence are maintained.

## Data Pipeline
FluidFXMovementSettings acts as a critical translation point, converting raw network bytes into a structured object that the game engine can use to configure player physics.

> Flow:
> Server Game State -> Serializer -> Network Packet -> Client ByteBuf -> **FluidFXMovementSettings.deserialize** -> FluidFXMovementSettings Instance -> Player Physics Controller -> Updated Movement Logic

