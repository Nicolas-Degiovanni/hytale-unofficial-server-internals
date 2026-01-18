---
description: Architectural reference for DayTexture
---

# DayTexture

**Package:** com.hypixel.hytale.server.core.asset.type.weather.config
**Type:** Data Model / POJO

## Definition
```java
// Signature
public class DayTexture {
```

## Architecture & Concepts
The DayTexture class is a simple data model object, often referred to as a Plain Old Java Object (POJO). Its primary architectural role is to serve as a structured, in-memory representation of a configuration entry that maps a specific day cycle index to a sky texture asset.

This class is a foundational component of Hytale's data-driven weather and sky rendering system. It does not contain any logic itself; instead, it acts as a schema for data that is loaded from external asset files.

The most critical component of this class is the public static final field **CODEC**. This `BuilderCodec` instance defines the contract for serialization and deserialization. It instructs the engine's asset loading system how to parse a data structure (e.g., a JSON object) and map its keys, "Day" and "Texture", to the corresponding fields of a new DayTexture instance. The inclusion of `CommonAssetValidator.TEXTURE_SKY` ensures that the provided texture path is validated against the engine's asset registry during the loading process, preventing runtime errors from invalid configuration.

## Lifecycle & Ownership
- **Creation:** DayTexture instances are created exclusively by the Hytale `Codec` system during the asset loading phase. The `BuilderCodec` invokes the protected no-argument constructor and subsequently populates the `day` and `texture` fields based on the source asset data.
- **Scope:** The lifetime of a DayTexture object is tightly coupled to its parent configuration object, such as a `WeatherProfile`. It persists in memory only as long as the parent asset is loaded.
- **Destruction:** The object is eligible for garbage collection once its containing asset is unloaded and all references to it are released. There is no manual destruction or cleanup mechanism.

## Internal State & Concurrency
- **State:** The class holds a simple, mutable state consisting of an integer and a String. While technically mutable, it is designed to be treated as immutable after the initial asset loading and deserialization process is complete. The game engine reads this state but does not modify it during runtime.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no internal synchronization. It is intended to be created and populated on a dedicated asset loading thread and subsequently read from the main game thread. Any external attempts to modify its state from multiple threads will lead to race conditions and undefined behavior.

## API Surface
The public API is minimal, consisting only of data accessors. The primary public contract is the static CODEC field, which is used by the asset system, not by typical game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDay() | int | O(1) | Returns the integer identifier for the day. |
| getTexture() | String | O(1) | Returns the asset path for the sky texture. |

## Integration Patterns

### Standard Usage
A developer does not instantiate or interact with this class directly. Instead, they define the data in an asset file (e.g., a weather profile JSON). Game systems then retrieve the loaded data from a higher-level manager.

```java
// A higher-level system retrieves the pre-loaded configuration
WeatherProfile profile = assetManager.get("hytale:weather.default");
List<DayTexture> dayTextures = profile.getDayTextures();

// The system then uses the data to drive rendering
for (DayTexture dt : dayTextures) {
    if (dt.getDay() == world.getTime().getDayIndex()) {
        skyRenderer.setSkyTexture(dt.getTexture());
        break;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new DayTexture()`. This bypasses the asset loading pipeline, the data-driven design, and critical validation steps. All weather data must be defined in asset files.
- **State Modification:** Do not modify the state of a DayTexture object after it has been loaded. While no public setters exist, using reflection or other means to change its fields at runtime can lead to visual inconsistencies and break assumptions made by the rendering engine.

## Data Pipeline
DayTexture is a destination for data flowing from asset files into the game engine's memory.

> Flow:
> Asset File (e.g., weather.json) -> AssetManager -> **DayTexture.CODEC** -> **DayTexture instance** -> WeatherProfile -> WeatherManager -> Sky Rendering System

