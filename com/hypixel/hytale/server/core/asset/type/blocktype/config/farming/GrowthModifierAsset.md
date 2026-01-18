---
description: Architectural reference for GrowthModifierAsset
---

# GrowthModifierAsset

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config.farming
**Type:** Data-Driven Component

## Definition
```java
// Signature
public abstract class GrowthModifierAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, GrowthModifierAsset>> {
```

## Architecture & Concepts

The GrowthModifierAsset is an abstract base class that forms the core of a data-driven system for calculating crop and plant growth rates on the server. It is not a concrete implementation itself, but rather a contract that defines how different environmental factors can influence the growth simulation.

This class acts as a bridge between static game configuration (defined in JSON asset files) and the dynamic world simulation. Concrete implementations of GrowthModifierAsset are not written in Java; they are defined entirely within asset packs. The engine uses the powerful `ABSTRACT_CODEC` to deserialize these JSON definitions into appropriate Java object instances at runtime. This decouples game design from engine programming, allowing designers to create complex growth rules without modifying server code.

All loaded GrowthModifierAsset instances are managed by a static, globally accessible AssetStore, which functions as a central registry. Game logic, such as the block update tick for a plant, queries this registry to retrieve the relevant modifiers and calculate the final growth multiplier.

## Lifecycle & Ownership

-   **Creation:** Instances are never created directly using the `new` keyword. They are instantiated by the Hytale AssetRegistry during server initialization or when asset packs are loaded. The `ABSTRACT_CODEC` reads a JSON configuration file and constructs the appropriate concrete subclass.
-   **Scope:** The lifecycle of a GrowthModifierAsset is tied to the server's global AssetStore. Once loaded from a data file, an instance persists for the entire duration of the server session. These objects should be treated as immutable configuration data.
-   **Destruction:** Instances are destroyed and garbage collected only when the AssetRegistry is shut down, which typically occurs during a full server stop. There is no mechanism for manual destruction or hot-reloading of individual assets.

## Internal State & Concurrency

-   **State:** The internal state, primarily the `id` and `modifier` fields, is considered immutable after deserialization. It represents static configuration loaded from disk.
-   **Thread Safety:** The object's state is safe to read from any thread due to its immutability. However, the primary logic method, `getCurrentGrowthMultiplier`, accepts mutable, thread-sensitive arguments like CommandBuffer and Ref<ChunkStore>.

    **Warning:** All calls to `getCurrentGrowthMultiplier` must be performed on the thread that owns the world chunk being simulated. Accessing ChunkStore references from other threads will lead to severe concurrency violations, data corruption, and server instability. The static `getAssetStore` method contains a lazy initializer which is only safe if first access occurs during the single-threaded server bootstrap phase.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) amortized | Retrieves the global, singleton-like store for all GrowthModifierAsset instances. |
| getAssetMap() | static DefaultAssetMap | O(1) amortized | A convenience method to directly access the map of all loaded assets from the store. |
| getCurrentGrowthMultiplier(...) | double | O(N) | Calculates the growth influence. Base implementation is O(1), but subclasses may perform complex world queries. |

## Integration Patterns

### Standard Usage

The intended pattern is to retrieve the central asset store, look up a specific modifier by its string identifier (as defined in a block's JSON configuration), and then invoke its logic as part of a larger simulation tick.

```java
// Within a server-side block update method
// Assume 'blockConfig' contains the ID of the modifier to use.
String modifierId = blockConfig.getGrowthModifierId();
DefaultAssetMap<String, GrowthModifierAsset> assetMap = GrowthModifierAsset.getAssetMap();
GrowthModifierAsset modifier = assetMap.get(modifierId);

if (modifier != null) {
    // Arguments (commandBuffer, refs, coordinates) are provided by the game loop.
    double growthRate = modifier.getCurrentGrowthMultiplier(
        commandBuffer, sectionRef, blockRef, x, y, z, isInitialTick
    );
    // ... apply growthRate to the block's state
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** This class is abstract and cannot be instantiated. Even for concrete subclasses (defined via JSON), direct instantiation is incorrect as it bypasses the asset loading and registration system.
-   **State Mutation:** Do not use reflection or other means to alter the `modifier` field after an asset has been loaded. These assets are designed to be immutable configuration.
-   **Cross-Thread Simulation:** Never call `getCurrentGrowthMultiplier` from a thread that does not have exclusive ownership of the CommandBuffer and ChunkStore references for the given coordinates.

## Data Pipeline

The flow of data for this system begins with a designer's configuration and ends with a calculated value in the game simulation.

> Flow:
> JSON Asset File -> AssetRegistry Deserialization -> **GrowthModifierAsset** instance in AssetStore -> Block Update Tick requests asset by ID -> `getCurrentGrowthMultiplier` is invoked -> Returns `double` multiplier -> Game Simulation updates block state

