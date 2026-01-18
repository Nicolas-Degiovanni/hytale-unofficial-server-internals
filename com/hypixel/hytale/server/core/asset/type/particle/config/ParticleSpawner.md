---
description: Architectural reference for ParticleSpawner
---

# ParticleSpawner

**Package:** com.hypixel.hytale.server.core.asset.type.particle.config
**Type:** Configuration Asset

## Definition
```java
// Signature
public class ParticleSpawner implements JsonAssetWithMap<String, DefaultAssetMap<String, ParticleSpawner>>, NetworkSerializable<com.hypixel.hytale.protocol.ParticleSpawner> {
```

## Architecture & Concepts

The ParticleSpawner class is a server-side data representation of a particle effect configuration. It is not a service or an active component; rather, it serves as an immutable blueprint that defines the behavior, appearance, and lifecycle of a specific particle effect within the game world.

Its primary architectural role is to bridge the gap between human-readable configuration files (JSON) and the engine's internal systems, including the network protocol. This is achieved through Hytale's powerful **Codec** system. A static AssetBuilderCodec, named CODEC, is defined within the class. This codec is responsible for deserializing a .particle JSON file into a fully-formed ParticleSpawner object.

A key feature of this architecture is support for asset inheritance. The codec's use of **appendInherited** allows particle definitions to extend and override properties from parent definitions, promoting reusability and simplifying the creation of complex effect variations.

Once loaded, a ParticleSpawner instance can be converted into a network-optimized packet via the NetworkSerializable interface. This allows the server to efficiently transmit the complete definition of a particle effect to clients, which then use that data to render the effect visually.

## Lifecycle & Ownership

-   **Creation:** ParticleSpawner instances are **exclusively** instantiated by the Hytale AssetStore during the server's asset loading phase. The static CODEC field dictates how JSON data is mapped to the object's fields. Direct instantiation by developers is an anti-pattern and will result in an unmanaged, incomplete object.
-   **Scope:** Once loaded by the AssetStore, a ParticleSpawner object is cached and persists for the entire server session. It is stored in a static, globally accessible registry, making it available to any system that requires it.
-   **Destruction:** Objects are garbage collected along with the AssetStore when the server shuts down. There is no manual destruction or cleanup process.

## Internal State & Concurrency

-   **State:** A ParticleSpawner object is mutable only during its initial creation by the asset codec. After it is published to the AssetStore, it should be treated as **effectively immutable**. All of its fields represent static configuration data.

    It contains one piece of mutable internal state: a SoftReference named cachedPacket. This field caches the generated network packet to avoid redundant object creation on subsequent calls to the toPacket method, acting as a performance optimization.

-   **Thread Safety:** This class is **not thread-safe**. The fields are not protected by synchronization primitives. The intended operational model is write-once (during asset loading on a single thread) and read-many (by multiple game logic threads).

    **WARNING:** The lazy initialization pattern in both getAssetStore and the caching mechanism in toPacket are not atomic. Concurrent calls on a cold system may result in minor inefficiencies (e.g., redundant object creation), but not data corruption. However, any external mutation of a ParticleSpawner object after it has been loaded will introduce severe race conditions and undefined behavior across the server.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, static registry for all ParticleSpawner assets. |
| toPacket() | com.hypixel.hytale.protocol.ParticleSpawner | O(N) first call, O(1) subsequent | Converts the asset into its network-serializable representation. The first call is more expensive as it builds the packet object. |
| getId() | String | O(1) | Returns the unique asset identifier, e.g., "hytale:my_effect". |

## Integration Patterns

### Standard Usage

The correct pattern is to retrieve a pre-loaded ParticleSpawner instance from the global AssetStore using its unique string identifier. This object is then typically used to spawn an effect in the world or converted to a packet for network transmission.

```java
// Retrieve the globally managed instance of a particle spawner asset
AssetStore<String, ParticleSpawner, ?> store = ParticleSpawner.getAssetStore();
ParticleSpawner fireEffect = store.get("hytale:fire_explosion");

if (fireEffect != null) {
    // Convert to a network packet to send to clients
    com.hypixel.hytale.protocol.ParticleSpawner packet = fireEffect.toPacket();
    networkManager.sendPacketToClients(packet);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ParticleSpawner()`. This bypasses the asset loading pipeline, the codec, and the asset inheritance system. The resulting object will be uninitialized and will cause NullPointerExceptions.
-   **Post-Load Mutation:** Do not modify the state of a ParticleSpawner object retrieved from the AssetStore. These objects are shared globally. Modifying one will affect every system using that particle effect, leading to unpredictable and difficult-to-debug visual artifacts.
-   **Assuming Existence:** Do not assume an asset exists. Always check for null after retrieving a spawner from the AssetStore, as the requested asset may be missing or may have failed to load.

## Data Pipeline

The flow of data from configuration file to client-side rendering is linear and unidirectional. The ParticleSpawner class is a critical stage in this pipeline, transforming static data into a network-ready format.

> Flow:
> .particle JSON File -> AssetStore Loader -> **ParticleSpawner.CODEC** -> In-Memory **ParticleSpawner** Instance -> toPacket() -> Network Layer -> Client Rendering Engine

