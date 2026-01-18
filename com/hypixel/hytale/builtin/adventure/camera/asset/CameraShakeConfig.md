---
description: Architectural reference for CameraShakeConfig
---

# CameraShakeConfig

**Package:** com.hypixel.hytale.builtin.adventure.camera.asset
**Type:** Data Model / Asset

## Definition
```java
// Signature
public class CameraShakeConfig implements NetworkSerializable<com.hypixel.hytale.protocol.CameraShakeConfig> {
```

## Architecture & Concepts
CameraShakeConfig is a passive data structure that defines the parameters for a camera shake effect. It is not an active engine component; rather, it serves as the in-memory representation of a camera shake asset, typically loaded from a JSON or equivalent configuration file.

The core of this class is the static `CODEC` field, which uses the Hytale `BuilderCodec` system. This codec is responsible for deserializing an asset file into a valid CameraShakeConfig object, applying validators, and constructing the nested object graph. This design pattern decouples the data definition (the asset file) from the engine's runtime representation (this class).

A critical architectural pattern is the separation between the asset model and the network protocol model. This class, `CameraShakeConfig`, is the high-level, editor-friendly asset definition. To be utilized by a game client, it must be converted into a more primitive, network-optimized format. The `NetworkSerializable` interface and the `toPacket` method serve as the bridge, transforming this configuration object into a `com.hypixel.hytale.protocol.CameraShakeConfig` packet payload for transmission.

The class is compositional, built from smaller configuration objects like `EasingConfig`, `OffsetNoise`, and `RotationNoise`. This allows content creators to define complex, layered effects by combining multiple noise sources for both translational and rotational camera movement.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Codec system when a corresponding asset is loaded, for example, via an `AssetManager`. Manual instantiation using the `new` keyword is an anti-pattern and will result in a misconfigured or empty object.
- **Scope:** The lifetime of a CameraShakeConfig instance is tied to its usage. If loaded as a shared asset, it may persist in a cache for the duration of a game session. If created for a one-off dynamic effect, it is short-lived.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The object's state is mutable only during its construction by the `BuilderCodec`. After deserialization is complete, it is treated as an immutable, read-only data container. All fields are `protected`, and no public setters are exposed, enforcing this contract.
- **Thread Safety:** This class is thread-safe for read operations. As a "read-mostly" configuration object, it can be safely shared and accessed by multiple systems on different threads *after* it has been fully constructed. It is **not** safe for concurrent modification, but the class design inherently prevents this.

## API Surface
The public contract is minimal, focused entirely on its role as a serializable data object.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.CameraShakeConfig | O(1) | Transforms the asset model into its corresponding network protocol object for client-server communication. This is the primary entry point for using the configuration. |

## Integration Patterns

### Standard Usage
The intended use is for a server-side system to load a predefined camera shake asset and trigger it on one or more clients. The system retrieves the asset, converts it to a packet, and dispatches it over the network.

```java
// A server-side system loads the config from an asset path
// (Conceptual example, actual API may vary)
CameraShakeConfig explosionShake = assetManager.load("hytale:camera_shakes/heavy_explosion.json");

// When an in-game event occurs, the config is converted to a packet and sent to the player
// This triggers the camera shake effect on the client renderer
player.getConnection().send(new S2CCameraShakePacket(explosionShake.toPacket()));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CameraShakeConfig()`. The default constructor exists for the codec system. Manual creation bypasses all field population, validation, and default value logic, resulting in a non-functional object.
- **State Mutation:** Do not attempt to modify the object's fields after it has been loaded. Camera shake assets are intended to be static data. If a dynamic shake is required, a new configuration should be built and sent.

## Data Pipeline
The flow of data from a content file to a rendered effect is a clear, multi-stage pipeline where CameraShakeConfig serves as the structured, in-memory representation.

> Flow:
> Asset File (JSON) -> Hytale Codec Deserializer -> **CameraShakeConfig** -> `toPacket()` -> Protocol Packet -> Network Layer -> Client-Side Effect System -> Camera Renderer

