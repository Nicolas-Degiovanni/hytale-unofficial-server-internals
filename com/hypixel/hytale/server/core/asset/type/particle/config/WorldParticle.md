---
description: Architectural reference for WorldParticle
---

# WorldParticle

**Package:** com.hypixel.hytale.server.core.asset.type.particle.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class WorldParticle implements NetworkSerializable<com.hypixel.hytale.protocol.WorldParticle> {
```

## Architecture & Concepts
The WorldParticle class is a passive data model that represents the configuration for a single particle effect to be spawned in the game world. It is not a manager or a service; its primary role is to act as a structured container for properties deserialized from server-side asset files, such as JSON or HOCON definitions.

The static **CODEC** field is the cornerstone of this class's design. It utilizes the engine's declarative Codec system to define the serialization and deserialization contract. This approach decouples the data model from the underlying storage format and centralizes validation logic. For instance, the codec ensures that the specified *systemId* refers to a valid and cached ParticleSystem before the object is even constructed, preventing runtime errors.

Its secondary but equally critical function is to serve as a bridge between the server's internal configuration and the client-facing network protocol. By implementing the NetworkSerializable interface, it provides a standardized `toPacket` method to translate its state into a raw `com.hypixel.hytale.protocol.WorldParticle` packet, ready for transmission to clients.

## Lifecycle & Ownership
- **Creation:** WorldParticle instances are almost exclusively created by the engine's Codec system during the asset loading phase. Game logic does not typically instantiate this class directly. Instead, a higher-level system (e.g., an animation controller, block behavior script) references an asset that contains a WorldParticle definition, triggering its deserialization.
- **Scope:** Transient and short-lived. An instance's lifetime is typically bound to the event that triggers the particle effect. It is created, converted into a network packet via `toPacket`, and then immediately becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** Mutable. The object's fields are populated after construction by the Codec system. While technically modifiable after creation, instances should be treated as effectively immutable once deserialized. They are designed to be read-only data carriers.
- **Thread Safety:** This class is **not thread-safe**. It is designed for use within a single thread, typically the main server game loop. Sharing instances across threads without external synchronization mechanisms is unsafe and will lead to unpredictable behavior.

## API Surface
The public API is minimal, consisting primarily of data accessors and the network serialization contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.WorldParticle | O(1) | Translates this configuration object into its network protocol equivalent for client transmission. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve a pre-configured WorldParticle instance from a loaded asset and use it to generate a network packet for spawning.

```java
// Assume 'loadedAsset' is an object deserialized from a config file
// that contains a WorldParticle definition.

WorldParticle particleConfig = loadedAsset.getParticleEffect();

// Convert the configuration into a transmittable network packet
com.hypixel.hytale.protocol.WorldParticle packet = particleConfig.toPacket();

// Broadcast the packet to relevant clients to render the effect
world.getPacketManager().sendToNearbyPlayers(position, packet);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid using `new WorldParticle()`. The static CODEC is the canonical way to create instances, as it ensures all validation rules are enforced. Manual creation bypasses critical checks, such as verifying the existence of the particle system ID.
- **State Reuse:** Do not cache and modify a single WorldParticle instance for multiple different effects. These objects are lightweight and should be treated as disposable. Reusing them can lead to state corruption and is not thread-safe.
- **Manual Serialization:** Do not attempt to write your own JSON or binary serializer for this class. The static CODEC is the single source of truth for its data contract and is guaranteed to be compatible with the engine's asset pipeline.

## Data Pipeline
The WorldParticle class is a key component in the pipeline that transforms a static asset definition into a dynamic, visible effect for the player.

> Flow:
> Asset File (e.g., JSON) -> Engine Asset Loader -> **WorldParticle.CODEC** -> **WorldParticle Instance** -> Game Logic Trigger -> `toPacket()` -> Network Layer -> Client Renderer

