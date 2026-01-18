---
description: Architectural reference for GatheringConfig
---

# GatheringConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Configuration Model / Transient

## Definition
```java
// Signature
public class GatheringConfig {
```

## Architecture & Concepts
The GatheringConfig class is a data model that represents a specific slice of server-side gameplay configuration. It is not a service or a manager, but rather a passive data structure that defines the outcomes of specific player gathering actions, such as attempting to break an unbreakable block or using an incorrect tool.

Its primary architectural role is to serve as a strongly-typed Java representation of configuration data defined in external asset files, likely JSON. The static final field named CODEC is the most critical component of this class. This `BuilderCodec` instance acts as the bridge between the raw asset data and the in-memory object, defining the exact rules for serialization and deserialization.

This class is a leaf node in the asset hierarchy. It is designed to be embedded within larger, more comprehensive asset definitions, such as those for blocks or items. The use of `appendInherited` within the CODEC definition indicates that these configurations can be part of a larger inheritance chain, allowing designers to define base gathering behaviors and override them in more specific assets.

## Lifecycle & Ownership
- **Creation:** A GatheringConfig instance is never created directly by game logic. It is instantiated and populated exclusively by the Hytale `codec` system during the asset loading phase. The static CODEC field is invoked by a higher-level asset manager when it encounters a corresponding data block in an asset file.
- **Scope:** The lifetime of a GatheringConfig object is strictly tied to the parent asset that contains it. It persists in memory as long as the parent asset is loaded and referenced by the server.
- **Destruction:** The object is eligible for garbage collection once its parent asset is unloaded and all references to it are released. It does not manage any native resources and has no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state consists of two `GatheringEffectsConfig` objects. While the fields are technically mutable, the object should be treated as **immutable** after the initial asset loading and deserialization process is complete. Its state is a direct reflection of the data on disk.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container. It is designed to be created and populated on an asset-loading thread and subsequently read by the main server game thread. Any concurrent modification would result in undefined behavior and data corruption. If access across multiple threads is required, synchronization must be handled externally.

## API Surface
The public contract is minimal, providing read-only access to the nested configuration objects.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getUnbreakableBlockConfig() | GatheringEffectsConfig | O(1) | Retrieves the configuration for effects triggered when a player attempts to break a block designated as unbreakable. |
| getIncorrectToolConfig() | GatheringEffectsConfig | O(1) | Retrieves the configuration for effects triggered when a player uses an incorrect tool for a specific gathering action. |

## Integration Patterns

### Standard Usage
This class is not meant to be retrieved from a service registry. It is accessed as a property of a larger, parent asset configuration. Game logic systems query this object to determine how to respond to player actions.

```java
// Pseudo-code demonstrating access via a parent asset
BlockConfig targetBlock = assetRegistry.get(BlockConfig.class, "hytale:stone");
GatheringConfig gatheringRules = targetBlock.getGatheringConfig();

// A game system checks if the player's tool is valid
if (!isToolCorrect(player.getHeldItem(), targetBlock)) {
    GatheringEffectsConfig effects = gatheringRules.getIncorrectToolConfig();
    // Trigger sounds, particles, or other feedback based on the effects config
    EffectPlayer.play(player, effects);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new GatheringConfig()`. This bypasses the asset pipeline and results in an object with default, empty values. Such an object will not reflect the intended game design stored in the asset files. All instances must be created via the CODEC deserialization process.
- **State Mutation:** Do not modify the state of a GatheringConfig object at runtime after it has been loaded. This breaks the principle that assets are the single source of truth for game data and can lead to inconsistent and unpredictable behavior for players.

## Data Pipeline
The GatheringConfig class is a destination point in the server's asset loading pipeline. It transforms serialized data from disk into a usable in-memory object for the game simulation.

> Flow:
> Asset File (JSON) -> Server AssetManager -> `BuilderCodec` Deserializer -> **GatheringConfig Instance** -> Game Logic (e.g., BlockInteractionSystem)

