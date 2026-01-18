---
description: Architectural reference for ReputationGameplayConfig
---

# ReputationGameplayConfig

**Package:** com.hypixel.hytale.builtin.adventure.reputation
**Type:** Configuration Model / Transient

## Definition
```java
// Signature
public class ReputationGameplayConfig {
```

## Architecture & Concepts
The ReputationGameplayConfig class is a passive data structure that defines the storage strategy for the player reputation system. It is not an active service or manager, but rather a configuration object deserialized from game assets.

This class serves as a typed, plugin-specific configuration payload within the broader GameplayConfig system. The engine uses the static CODEC field, a BuilderCodec, to automatically instantiate and populate this object from configuration files (e.g., adventure mode scripts). This pattern decouples the reputation system's logic from the hardcoded configuration values, allowing game designers to control its behavior on a per-world or per-game-mode basis.

Its primary architectural function is to inform the reputation persistence layer whether a player's reputation is global (PerPlayer) or scoped to a specific game instance (PerWorld).

## Lifecycle & Ownership
- **Creation:** Instances are materialized by the Hytale codec framework when a parent GameplayConfig is loaded from an asset file. The static CODEC field dictates the deserialization rules. A static default instance, DEFAULT_REPUTATION_GAMEPLAY_CONFIG, is also created at class-load time to serve as a fallback.
- **Scope:** The lifetime of a ReputationGameplayConfig instance is strictly bound to its parent GameplayConfig object. It persists in memory as long as the game mode or world it belongs to is active.
- **Destruction:** The object is eligible for garbage collection once its parent GameplayConfig is unloaded or otherwise dereferenced. It has no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The internal state consists of a single field, reputationStorageType. While the field is not declared final, the object is designed to be *effectively immutable* after its initial creation by the codec system. There are no public methods to mutate its state post-instantiation.
- **Thread Safety:** This class is not inherently thread-safe. However, due to its effectively immutable nature, it is safe to read from multiple threads *after* it has been fully constructed and published by the configuration loading system. Any external mutation would be a violation of its design and could lead to unpredictable behavior.

## API Surface
The public API is composed entirely of static factory methods and a single instance-level accessor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(GameplayConfig) | static ReputationGameplayConfig | O(1) | Retrieves the reputation config from a parent GameplayConfig. May return null. |
| getOrDefault(GameplayConfig) | static ReputationGameplayConfig | O(1) | Retrieves the reputation config, returning a default instance if not found. This is the preferred accessor. |
| getReputationStorageType() | ReputationStorageType | O(1) | Returns the configured storage strategy for the reputation system. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve this configuration from a broader GameplayConfig context, typically available within server-side game logic. Always prefer the default-providing method to prevent NullPointerExceptions.

```java
// Obtain the active gameplay configuration for the current context
GameplayConfig gameplayConfig = world.getGameplayConfig();

// Safely retrieve the reputation settings
ReputationGameplayConfig repConfig = ReputationGameplayConfig.getOrDefault(gameplayConfig);
ReputationStorageType storageType = repConfig.getReputationStorageType();

if (storageType == ReputationStorageType.PerWorld) {
    // Execute logic specific to world-based reputation
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new ReputationGameplayConfig()`. This bypasses the entire configuration system and will yield a default object that does not reflect the actual settings loaded from game assets.
- **Assuming Non-Null:** Do not use the `get()` method without a null check. If the configuration is absent from the game assets, it will return null, leading to runtime errors.

## Data Pipeline
This class sits at the end of the configuration loading pipeline and at the beginning of the reputation system's decision-making process.

> Flow:
> Game Asset File (e.g., world.json) -> Hytale Codec Engine -> GameplayConfig -> **ReputationGameplayConfig** -> Reputation System Logic

