---
description: Architectural reference for ParticleAnimationFrame
---

# ParticleAnimationFrame

**Package:** com.hypixel.hytale.server.core.asset.type.particle.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class ParticleAnimationFrame implements NetworkSerializable<com.hypixel.hytale.protocol.ParticleAnimationFrame> {
```

## Architecture & Concepts
The ParticleAnimationFrame class is a static data model that represents a single keyframe within a particle effect's animation timeline. It is not a dynamic, in-world entity but rather a blueprint used by the particle simulation system. This class serves as a server-side representation of configuration defined in particle asset files (e.g., JSON).

Its primary architectural role is to be deserialized by the engine's universal `Codec` system. The static `CODEC` field is the contract that allows the engine to translate raw configuration data into a strongly-typed Java object. This object encapsulates all visual properties of a particle—such as scale, rotation, and color—at a specific point in its life.

Once loaded, it is used by the server to construct network packets that instruct clients on how to render the particle effect. The implementation of the NetworkSerializable interface, specifically the `toPacket` method, is the bridge between the server's asset representation and the client's rendering protocol.

## Lifecycle & Ownership
- **Creation:** Instances of ParticleAnimationFrame are created exclusively by the Hytale `Codec` system during the server's asset loading phase. The engine reads a particle definition file, and the `ParticleAnimationFrame.CODEC` is invoked to deserialize the relevant section into a new object. Manual instantiation is strongly discouraged.
- **Scope:** The object's lifetime is bound to its parent particle effect asset. It persists in memory as long as the parent asset is loaded and referenced by the AssetManager. It is effectively a static, read-only piece of configuration data for the duration of its life.
- **Destruction:** The object is eligible for garbage collection once the parent particle effect asset is unloaded from the server, typically during a zone change, server shutdown, or when assets are no longer needed.

## Internal State & Concurrency
- **State:** The internal state is mutable upon construction but should be treated as **effectively immutable** after deserialization is complete. The fields (frameIndex, scale, etc.) are populated once by the `CODEC` and are not intended to be changed during runtime. This "load-once, read-many" pattern is central to the asset system's design.
- **Thread Safety:** This class is inherently thread-safe for read operations. As a simple data holder with no internal logic that modifies its own state, it can be safely read by multiple threads (e.g., game logic thread, networking thread) without synchronization, provided the immutability contract is respected.

## API Surface
The public API is minimal, consisting primarily of accessors for its configured properties and the network serialization method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFrameIndex() | Range | O(1) | Returns the time range for which this keyframe is active. |
| getScale() | RangeVector2f | O(1) | Returns the scale range of the particle during this frame. |
| getRotation() | RangeVector3f | O(1) | Returns the rotation range of the particle during this frame. |
| getColor() | Color | O(1) | Returns the color of the particle during this frame. |
| getOpacity() | float | O(1) | Returns the opacity of the particle. A value of -1 indicates unassigned. |
| toPacket() | ParticleAnimationFrame | O(1) | Creates and returns a network protocol-compatible representation of this object. |

## Integration Patterns

### Standard Usage
Direct interaction with this class in Java is uncommon. The standard "usage" is declarative, by defining its properties within a particle asset file. The engine handles the loading and processing.

A developer would typically define a structure like this in a particle JSON file:

```json
// Example: my_effect.json (particle definition)
{
  "animation": {
    "frames": [
      {
        "FrameIndex": [0.0, 0.1],
        "Scale": [[1.0, 1.0], [1.2, 1.2]],
        "Color": "FFFFFF",
        "Opacity": 1.0
      },
      {
        "FrameIndex": [0.9, 1.0],
        "Scale": [[0.2, 0.2], [0.0, 0.0]],
        "Color": "FF8800",
        "Opacity": 0.0
      }
    ]
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new ParticleAnimationFrame()`. The `protected` default constructor is reserved for the `Codec` system. Manually creating these objects bypasses the asset management system and can lead to data that is not synchronized with the rest of the engine.
- **State Mutation:** Do not modify the fields of a ParticleAnimationFrame after it has been loaded by the engine. Assets are often shared resources, and mutating one instance can cause unpredictable visual artifacts and race conditions across the entire server.

## Data Pipeline
The ParticleAnimationFrame is a critical link in the chain that transforms a configuration file on disk into a visual effect on a player's screen.

> Flow:
> Particle Config File (JSON) -> Codec Deserializer -> **ParticleAnimationFrame** (Server Memory) -> toPacket() -> Network Packet -> Client Renderer

