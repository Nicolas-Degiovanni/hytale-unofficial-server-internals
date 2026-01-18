---
description: Architectural reference for BlockParticleSet
---

# BlockParticleSet

**Package:** com.hypixel.hytale.server.core.asset.type.blockparticle.config
**Type:** Data Asset

## Definition
```java
// Signature
public class BlockParticleSet
   implements JsonAssetWithMap<String, DefaultAssetMap<String, BlockParticleSet>>,
   NetworkSerializable<com.hypixel.hytale.protocol.BlockParticleSet> {
```

## Architecture & Concepts
The BlockParticleSet class is a data-driven configuration asset that defines a collection of particle effects associated with specific block-related events, such as breaking, placing, or walking on a block. It serves as a critical link between the game's static asset definitions (typically JSON files) and the runtime particle rendering and networking systems.

Its primary architectural role is to be a deserialized, in-memory representation of a particle configuration. The static **CODEC** field, an instance of AssetBuilderCodec, dictates how this object is constructed from raw data. This codec is central to the class's design, enabling sophisticated features like property inheritance. A specific BlockParticleSet can inherit default values for color, scale, and offsets from a parent definition, significantly reducing asset duplication and improving maintainability.

By implementing NetworkSerializable, this class can be efficiently converted into a lightweight network packet for transmission to clients, ensuring that visual effects are synchronized across the server and all connected players.

## Lifecycle & Ownership
- **Creation:** BlockParticleSet instances are created exclusively by the Hytale **AssetStore** during the engine's asset loading phase at startup. The static CODEC is invoked by the asset management system to parse JSON definitions and construct the corresponding Java objects. Direct instantiation by developers is an anti-pattern and will lead to unmanaged, non-functional objects.

- **Scope:** An instance of BlockParticleSet persists for the entire application session. All loaded instances are owned and managed by the central **AssetRegistry** via the BlockParticleSet specific AssetStore. They are effectively global, read-only resources.

- **Destruction:** Instances are garbage collected when the AssetStore is cleared, which typically occurs only during application shutdown or a full asset hot-reload.

## Internal State & Concurrency
- **State:** The object's state is **effectively immutable** after its initial creation by the asset loader. Its fields, such as color, scale, and the map of particle system IDs, are intended to be read-only configuration data. However, the class contains one piece of mutable state: a `SoftReference` to a cached network packet (`cachedPacket`). This is a performance optimization to avoid repeated allocation of the protocol object.

- **Thread Safety:** This class is **not thread-safe**. While the primary configuration data is safe to read from any thread due to its immutability, the `toPacket` method performs a non-atomic check-then-act on the `cachedPacket` field.

    **WARNING:** Concurrent calls to `toPacket` from multiple threads can result in a race condition, where multiple packet objects are created unnecessarily. If this class is used in a multi-threaded context (e.g., multiple network threads preparing packets), access to `toPacket` must be externally synchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | AssetStore | O(1) | Static accessor for the global repository of all BlockParticleSet assets. |
| toPacket() | BlockParticleSet | O(1) amortized | Converts the asset into its network protocol representation. Caches the result in a SoftReference. **Not thread-safe.** |
| getParticleSystemIds() | Map | O(1) | Returns the map linking a BlockParticleEvent (e.g., BREAK) to a particle system asset ID. |

## Integration Patterns

### Standard Usage
A game system should retrieve a BlockParticleSet instance from the global AssetStore using its unique asset key. It should never be instantiated directly.

```java
// Retrieve the central store for these assets
AssetStore<String, BlockParticleSet, ?> store = BlockParticleSet.getAssetStore();

// Get a specific particle set by its ID (e.g., "hytale:stone_particles")
BlockParticleSet stoneParticles = store.get("hytale:stone_particles");

if (stoneParticles != null) {
    // Get the specific particle system to spawn for a "break" event
    String particleSystemId = stoneParticles.getParticleSystemIds().get(BlockParticleEvent.BREAK);

    // Trigger the particle system using the retrieved ID
    particleManager.spawn(particleSystemId);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockParticleSet()`. This bypasses the asset loading pipeline, the central registry, and the inheritance system. The resulting object will be uninitialized and disconnected from the game engine.

- **State Mutation:** Do not attempt to modify the fields of a BlockParticleSet after it has been loaded. These objects are designed as read-only configurations.

- **Concurrent Packet Creation:** Do not call `toPacket()` from multiple threads simultaneously without external locking. This can lead to race conditions and unpredictable behavior.

## Data Pipeline
BlockParticleSet exists at two key points in the engine's data flow: asset loading and network serialization.

> **Asset Loading Flow:**
> JSON File on Disk -> AssetStore Loader -> **BlockParticleSet.CODEC** -> In-Memory **BlockParticleSet** Instance -> AssetRegistry

> **Network Serialization Flow:**
> Server Game Event -> AssetRegistry Lookup -> **BlockParticleSet** Instance -> `toPacket()` -> `protocol.BlockParticleSet` -> Network Encoder -> TCP/UDP Packet -> Client

