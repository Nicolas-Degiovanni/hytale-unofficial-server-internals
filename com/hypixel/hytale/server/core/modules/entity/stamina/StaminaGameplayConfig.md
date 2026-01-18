---
description: Architectural reference for StaminaGameplayConfig
---

# StaminaGameplayConfig

**Package:** com.hypixel.hytale.server.core.modules.entity.stamina
**Type:** Configuration Model

## Definition
```java
// Signature
public class StaminaGameplayConfig {

   // Nested Configuration
   public static class SprintRegenDelayConfig {
      // ...
   }
}
```

## Architecture & Concepts

The StaminaGameplayConfig class is a data model that represents server-side gameplay configuration for entity stamina mechanics. It is not an active service or manager; instead, it serves as a strongly-typed, in-memory representation of settings loaded from external asset files, likely in JSON format.

This class is a cornerstone of Hytale's data-driven design. Its primary architectural feature is the static **CODEC** field, an instance of BuilderCodec. This codec defines the schema, validation rules, and deserialization logic for the configuration. It acts as the bridge between raw data in an asset file and a validated, type-safe Java object that the game engine can use directly.

The core responsibility of this object is to hold parameters that govern stamina regeneration delay after sprinting. It achieves this through its nested SprintRegenDelayConfig class, which links a specific EntityStat (identified by a string ID) to a numerical value. During the asset loading process, the codec's **afterDecode** hook resolves the string-based stat ID into a more performant integer index for fast runtime lookups, a critical optimization for game logic.

## Lifecycle & Ownership

-   **Creation:** An instance of StaminaGameplayConfig is never created directly using its constructor in game logic. It is exclusively instantiated by the Hytale **Codec** system during the server's asset loading phase. The codec populates the object's fields from a corresponding data file.
-   **Scope:** The object's lifetime is tied to the scope of the configuration it represents. Typically, it is loaded once on server or world initialization and held by a central configuration registry or the primary entity stamina module. It persists for the duration of the game session.
-   **Destruction:** The object is a plain Java object with no explicit destruction logic. It is marked for garbage collection when the server or world instance that owns its configuration is shut down and all references to it are released.

## Internal State & Concurrency

-   **State:** The state of a StaminaGameplayConfig instance should be considered **effectively immutable** after its creation by the codec system. While its fields are not declared final, the design pattern dictates that it is a read-only data container once loaded. The only mutation occurs within the codec's `afterDecode` hook, which is a controlled part of the initialization pipeline.

-   **Thread Safety:** The class is **conditionally thread-safe**. As a simple data holder, it is perfectly safe to read from multiple threads simultaneously, provided that initialization is complete. It contains no locks or synchronization primitives. Writing to its fields after initialization is not a thread-safe operation and is a violation of its intended use.

## API Surface

The public API is minimal, exposing only read-only access to the configured data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSprintRegenDelay() | SprintRegenDelayConfig | O(1) | Returns the nested configuration object for sprint regeneration delay. |
| SprintRegenDelayConfig.getIndex() | int | O(1) | Returns the pre-calculated integer index for the associated entity statistic. |
| SprintRegenDelayConfig.getValue() | float | O(1) | Returns the configured value for the stamina regeneration delay. |

## Integration Patterns

### Standard Usage

The class is intended to be deserialized by a higher-level asset or configuration manager. Game systems then retrieve the populated instance to query gameplay values.

```java
// Pseudo-code for loading and using the configuration
// In a configuration loading system:
StaminaGameplayConfig staminaConfig = assetManager.load("stamina.json", StaminaGameplayConfig.CODEC);
configRegistry.register(staminaConfig);

// In the EntityStaminaModule:
StaminaGameplayConfig config = configRegistry.get(StaminaGameplayConfig.class);
float delayValue = config.getSprintRegenDelay().getValue();
int statIndex = config.getSprintRegenDelay().getIndex();
player.applyStatModifier(statIndex, delayValue);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance using `new StaminaGameplayConfig()`. This will bypass the codec, resulting in an object with null fields. Any attempt to access its data will cause a NullPointerException. The object is invalid unless created via its static CODEC.
-   **Runtime Mutation:** Do not modify the fields of a loaded StaminaGameplayConfig object at runtime. This breaks the principle of configuration-as-data and can lead to inconsistent behavior across different game systems that hold a reference to the same shared instance. Configuration should be reloaded, not mutated in-place.

## Data Pipeline

The StaminaGameplayConfig class is a key component in the data pipeline that transforms static asset files into active gameplay mechanics.

> Flow:
> Asset File (e.g., stamina.json) -> Server Asset Loader -> **StaminaGameplayConfig.CODEC** (Deserialization & Validation) -> **StaminaGameplayConfig Instance** -> Entity Stamina Module -> Live Gameplay Logic<ctrl63>

