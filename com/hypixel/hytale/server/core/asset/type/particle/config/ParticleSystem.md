---
description: Architectural reference for ParticleSystem
---

# ParticleSystem

**Package:** com.hypixel.hytale.server.core.asset.type.particle.config
**Type:** Asset Data Model

## Definition
```java
// Signature
public class ParticleSystem
   implements JsonAssetWithMap<String, DefaultAssetMap<String, ParticleSystem>>,
   NetworkSerializable<com.hypixel.hytale.protocol.ParticleSystem> {
```

## Architecture & Concepts
The ParticleSystem class is a server-side, in-memory representation of a particle system asset. It is a fundamental data model within the Hytale Asset System, acting as the bridge between declarative JSON configuration files on disk and the runtime objects used by the game engine.

Its structure is defined by a static **AssetBuilderCodec**, a powerful deserialization engine that maps JSON properties to class fields. This codec-driven approach allows for complex features like property inheritance from parent assets, declarative validation rules, and metadata for tooling, such as UI icons.

Functionally, a ParticleSystem defines the complete behavior of a visual effect, including its lifespan, rendering bounds, and a collection of **ParticleSpawnerGroup** objects that control the emission of individual particles. It also implements the NetworkSerializable interface, enabling it to be efficiently converted into a network-optimized Data Transfer Object (DTO) for synchronization with game clients.

## Lifecycle & Ownership
- **Creation:** A ParticleSystem instance is created exclusively by the **AssetStore** during the server's asset loading phase. The static CODEC field is invoked by the asset pipeline to deserialize a corresponding JSON file and construct the object. Direct instantiation by developers is a critical anti-pattern.
- **Scope:** Application-scoped. Once loaded, a ParticleSystem instance is registered in the static ASSET_STORE and persists for the entire lifetime of the server process. It is treated as a read-only configuration object post-initialization.
- **Destruction:** The object is eligible for garbage collection only when the server shuts down and the central AssetRegistry is cleared.

## Internal State & Concurrency
- **State:** The object's state is effectively immutable after being loaded from its asset file. All definitional properties like lifeSpan, spawners, and cullDistance are set once during deserialization. The only mutable state is the private **cachedPacket** field, which is a performance optimization.
- **Thread Safety:** The class is **conditionally thread-safe**. All configuration fields (id, lifeSpan, spawners, etc.) are safe to read from any thread after the asset loading phase is complete. The toPacket method, which manages the internal cache, is not strictly atomic but is safe for concurrent use. In a race condition, multiple threads may regenerate the network packet, with the last one written to the SoftReference being used. This is a benign race that only results in minor redundant computation.

## API Surface
The public API is designed for retrieving configuration data and converting the asset for network transmission.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, singleton asset store for all ParticleSystem assets. |
| getAssetMap() | static DefaultAssetMap | O(1) | A convenience method to get the underlying map of all loaded particle systems. |
| toPacket() | com.hypixel.hytale.protocol.ParticleSystem | O(N) / O(1) | Converts the asset into a network DTO. Complexity is O(N) on first call (N = spawners), and O(1) on subsequent calls if the cached packet is available. |
| getId() | String | O(1) | Returns the unique asset key, such as *hytale:fire_large*. |
| getSpawners() | ParticleSpawnerGroup[] | O(1) | Returns the array of spawner groups that define the particle emission behavior. |

## Integration Patterns

### Standard Usage
A ParticleSystem should always be retrieved from the central asset repository using its unique string identifier. Never store a direct reference if it can be avoided; prefer to store the ID and re-query the asset map.

```java
// Correctly retrieve a managed ParticleSystem instance
DefaultAssetMap<String, ParticleSystem> particleSystems = ParticleSystem.getAssetMap();
ParticleSystem fireEffect = particleSystems.get("hytale:fire_large");

if (fireEffect != null) {
    // Convert to a network packet to send to a client
    com.hypixel.hytale.protocol.ParticleSystem packet = fireEffect.toPacket();
    // networkManager.send(player, packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ParticleSystem()`. This creates an unmanaged, uninitialized object that is not registered with the game's asset system. It will cause NullPointerExceptions and inconsistent behavior.
- **State Modification:** Do not attempt to modify the fields of a ParticleSystem instance after it has been loaded. These objects are shared across the entire server and should be treated as immutable. Modifying one will affect every part of the game that uses it.
- **Cache Dependency:** Do not write code that assumes the result of `toPacket` is always physically the same object. The internal cache uses a SoftReference, which allows the Garbage Collector to reclaim the memory under pressure. The `toPacket` method will transparently regenerate it, but logic based on reference equality (==) will fail.

## Data Pipeline
The ParticleSystem class is a key stage in the asset-to-network pipeline for visual effects.

> Flow:
> JSON file on disk (`hytale/assets/hytale/particles/fire_large.json`) -> Server Asset Loader -> **AssetBuilderCodec** (Deserialization) -> **ParticleSystem instance** (In-Memory) -> `toPacket()` call -> `com.hypixel.hytale.protocol.ParticleSystem` (Network DTO) -> Server Network Layer -> Game Client Renderer

