---
description: Architectural reference for NoiseConfig
---

# NoiseConfig

**Package:** com.hypixel.hytale.builtin.adventure.camera.asset
**Type:** Asset Model / Data Transfer Object

## Definition
```java
// Signature
public class NoiseConfig implements NetworkSerializable<com.hypixel.hytale.protocol.NoiseConfig> {
```

## Architecture & Concepts
The NoiseConfig class is a data-driven asset model that defines the parameters for a procedural noise generator. Its primary application within the engine is to configure camera effects, such as shaking or subtle idle movements, to create a more dynamic and immersive player experience.

This class is a cornerstone of Hytale's serialization framework. It is not intended for direct instantiation in game logic. Instead, its structure is defined by a static **BuilderCodec**, which acts as a schema for deserializing configuration from asset files (e.g., JSON). This approach allows designers and developers to define complex camera behaviors in data files without modifying engine code.

Furthermore, by implementing the NetworkSerializable interface, NoiseConfig serves a dual purpose. It acts as the canonical representation for noise settings on the server, which can then be efficiently converted into a network-optimized packet object and synchronized with clients. This ensures that all players experience identical, server-authoritative camera effects.

## Lifecycle & Ownership
- **Creation:** Instances of NoiseConfig are created exclusively by the engine's serialization system via the static **CODEC** field. This occurs when a parent asset, such as a camera profile, is loaded from disk.
- **Scope:** The lifetime of a NoiseConfig instance is bound to the parent asset that contains it. It persists in memory as long as the parent asset is loaded and referenced.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for destruction once its parent asset is unloaded and all references to it are released. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** NoiseConfig is a **mutable** data container. Its fields are populated during the deserialization process. While technically mutable, it should be treated as an immutable object after initialization is complete. Modifying its state at runtime is an anti-pattern and can lead to desynchronization or unpredictable behavior.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. It is designed to be populated on a background loading thread and subsequently read from the main game thread. Unsynchronized access from multiple threads will result in undefined behavior.

## API Surface
The public API is minimal, focusing on its role as a data object and its network serialization contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.NoiseConfig | O(1) | Converts this asset model into its corresponding network packet DTO for client-server synchronization. |

## Integration Patterns

### Standard Usage
NoiseConfig objects are defined in external asset files and loaded by the engine. Game logic retrieves the fully-formed object from a parent asset and passes it to a relevant system, such as a camera controller.

```java
// A developer defines the noise configuration within a larger asset file.
// The engine's AssetManager handles loading and deserialization.

// Retrieve the parent asset, which now contains the populated NoiseConfig.
CameraProfile profile = assetManager.get("data/camera/profiles/player_sprint.json");
NoiseConfig sprintShake = profile.getShakeNoise();

// The fully configured object is then passed to the camera system.
cameraSystem.applyShake(sprintShake);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new NoiseConfig()`. This bypasses the codec, which is responsible for applying default values, validation, and proper field initialization from data files.
- **Runtime Mutation:** Do not modify the fields of a NoiseConfig object after it has been loaded and is in use by a system. This can break the "source of truth" principle and cause visual inconsistencies or network desynchronization. If a different behavior is needed, a new asset should be loaded.

## Data Pipeline
The class serves as a critical link between static game data, the server, and the client's rendering engine.

> Flow:
> Asset File (JSON) -> Engine Deserializer (using **CODEC**) -> **NoiseConfig Instance** -> toPacket() -> Server Network Layer -> Client -> Camera System

---

# NoiseConfig.ClampConfig

**Package:** com.hypixel.hytale.builtin.adventure.camera.asset
**Type:** Asset Model / Data Transfer Object

## Definition
```java
// Signature
public static class ClampConfig implements NetworkSerializable<com.hypixel.hytale.protocol.ClampConfig> {
```

## Architecture & Concepts
ClampConfig is a nested component of NoiseConfig, responsible for defining the output range of the noise function. It allows designers to restrict the noise values to a specific minimum and maximum, and optionally re-normalize the output back to a standard -1 to 1 range.

Like its parent, ClampConfig is a schema-defined data object built for serialization. Its **BuilderCodec** defines validation rules, such as ensuring the min and max values are within a sensible range, and includes post-processing logic to guarantee a valid state (min is always less than or equal to max). This encapsulates complex configuration logic away from the core camera systems.

Its implementation of NetworkSerializable ensures that these precise clamping and normalization rules are perfectly synchronized from the server to all connected clients.

## Lifecycle & Ownership
- **Creation:** Instantiated by the parent NoiseConfig's **BuilderCodec** during the asset deserialization process. It is never created independently.
- **Scope:** The lifecycle of a ClampConfig instance is strictly identical to its parent NoiseConfig object. It cannot exist on its own.
- **Destruction:** It is garbage collected when its parent NoiseConfig is destroyed.

## Internal State & Concurrency
- **State:** **Mutable**, but should be treated as immutable post-initialization. The codec may adjust the min and max fields after decoding to ensure logical consistency.
- **Thread Safety:** **Not thread-safe**. All access should be synchronized externally or confined to a single thread after the initial asset loading is complete.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.ClampConfig | O(1) | Converts this clamping configuration into its network packet DTO. |

