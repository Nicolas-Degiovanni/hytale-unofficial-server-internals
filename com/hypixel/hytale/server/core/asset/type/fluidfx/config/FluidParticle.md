---
description: Architectural reference for FluidParticle
---

# FluidParticle

**Package:** com.hypixel.hytale.server.core.asset.type.fluidfx.config
**Type:** Configuration Model

## Definition
```java
// Signature
public class FluidParticle implements NetworkSerializable<com.hypixel.hytale.protocol.FluidParticle> {
```

## Architecture & Concepts
The FluidParticle class is a server-side data model that defines the properties of a particle system associated with a fluid. It is not a live, in-world particle entity, but rather the static configuration or template from which such particles are spawned.

Its primary architectural role is to serve as a deserialization target for asset files, typically JSON or a similar format. The static **CODEC** field is central to this design, providing a single, declarative source of truth for serialization, deserialization, documentation, and validation logic. This codec-driven approach is a core pattern in the Hytale engine for managing game data.

By implementing the NetworkSerializable interface, this class acts as a bridge between the server's internal asset configuration and the client-facing network protocol. The `toPacket` method handles the transformation into a lightweight, network-optimized Data Transfer Object (DTO).

## Lifecycle & Ownership
- **Creation:** FluidParticle instances are created exclusively by the engine's asset loading system during server bootstrap. The static `CODEC` field is used to parse a corresponding definition from a fluid asset file and construct the object. Manual instantiation is a design violation.
- **Scope:** The object's lifetime is tied to the asset it was loaded from. It persists in memory for the entire server session, typically as part of a larger parent configuration object (e.g., a `Fluid` definition).
- **Destruction:** Instances are marked for garbage collection when the server shuts down or performs a full asset reload, at which point all references to the old configuration are dropped.

## Internal State & Concurrency
- **State:** This object is **effectively immutable**. While its fields are not declared final, the design pattern of loading from static assets implies they are never modified after initialization. The single piece of mutable state is the `cachedPacket` field, which is a performance optimization.
- **Thread Safety:** The class is **conditionally thread-safe**. It is safe for concurrent reads from multiple threads, which is the standard operational use case. Direct mutation is not thread-safe and violates design intent.

    **Warning:** The `toPacket` method contains a benign race condition in its caching logic. If multiple threads call it simultaneously on a cold cache, more than one network packet object may be created. However, the state of these objects will be identical, and the performance impact is negligible. This trade-off avoids the overhead of synchronization locks.

## API Surface
The public API is minimal, focusing on data access and network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toPacket() | com.hypixel.hytale.protocol.FluidParticle | O(1) | Converts the configuration object into its network-transferable representation. Utilizes an internal SoftReference cache for performance. |
| getSystemId() | String | O(1) | Returns the asset identifier for the particle system to be spawned. |
| getColor() | Color | O(1) | Returns the default color tint for the particle. |
| getScale() | float | O(1) | Returns the scale multiplier for the particle system. |

## Integration Patterns

### Standard Usage
This object is not intended for direct manipulation. The engine retrieves it from a parent configuration object (e.g., a Fluid) and uses its properties to inform the particle rendering system. The `toPacket` method is invoked transparently by the network layer when serializing game state for clients.

```java
// Example of how the engine might use this object internally
Fluid fluidDef = AssetManager.get("hytale:water");
FluidParticle particleConfig = fluidDef.getSplashParticle();

// The engine's particle spawner would then use these properties
ParticleSpawner.spawn(
    particleConfig.getSystemId(),
    position,
    particleConfig.getColor(),
    particleConfig.getScale()
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new FluidParticle()`. All configuration must be loaded from asset files via the `CODEC` to ensure validation and system-wide consistency.
- **Post-Load Mutation:** Do not modify the fields of a FluidParticle after it has been loaded by the asset manager. This breaks the assumption of immutability and can lead to unpredictable behavior across the server.
- **Bypassing the Packet Cache:** Do not attempt to manually create `com.hypixel.hytale.protocol.FluidParticle` instances from this object. Always use the `toPacket` method to leverage the caching mechanism.

## Data Pipeline
The FluidParticle class is a key step in the pipeline that transforms static asset data on disk into dynamic visual effects on the client's screen.

> Flow:
> Asset File (`.json`) -> Server AssetLoader using `FluidParticle.CODEC` -> **FluidParticle** (In-Memory Config) -> `toPacket()` -> `protocol.FluidParticle` (Network DTO) -> Network Encoder -> Client Particle System

