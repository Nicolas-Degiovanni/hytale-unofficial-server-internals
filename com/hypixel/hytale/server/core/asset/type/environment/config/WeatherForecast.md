---
description: Architectural reference for WeatherForecast
---

# WeatherForecast

**Package:** com.hypixel.hytale.server.core.asset.type.environment.config
**Type:** Data Model / Transient

## Definition
```java
// Signature
public class WeatherForecast implements IWeightedElement {
```

## Architecture & Concepts
The WeatherForecast class is a lightweight data model that represents a single, weighted possibility for a weather event within a larger environmental configuration, such as a biome. It is not a service or manager, but rather a passive data structure loaded directly from server asset files.

Its primary role is to link a specific Weather asset, identified by a string ID, to a numerical weight. This allows game systems to perform probabilistic selections. For example, a biome can define a list of WeatherForecasts to model its climate, where "Rain" might have a higher weight than "Snow".

A key architectural feature is the post-deserialization processing step. The string-based `weatherId` is resolved into a more performant integer-based `weatherIndex`. This is a critical optimization that avoids repeated, expensive string-to-index lookups during the main game loop. This conversion is handled automatically by the `CODEC` system via the `afterDecode` hook.

## Lifecycle & Ownership
- **Creation:** Instances are exclusively created and hydrated by the Hytale `Codec` system during the server's asset loading phase. They are deserialized from configuration files (e.g., biome definitions) that contain weather forecast lists. Manual instantiation is a critical anti-pattern.
- **Scope:** An instance of WeatherForecast has its lifetime scoped to its parent configuration object. If it is part of a Biome's asset definition, it will persist in memory as long as that Biome asset is loaded.
- **Destruction:** The object is marked for garbage collection when its parent asset is unloaded from memory. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** The object holds configuration state (`weatherId`, `weight`) which should be considered immutable after initial loading. It also contains derived, transient state (`weatherIndex`) which is populated once during the `afterDecode` lifecycle hook.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be instantiated and processed on the main server thread during asset loading. Once the `processConfig` method has completed, the object becomes effectively immutable and can be safely *read* by multiple threads. Any external modification to its state after initialization will lead to undefined behavior and is strictly forbidden.

## API Surface
The public API is minimal, designed for read-only access to the processed configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWeatherId() | String | O(1) | Returns the raw string identifier for the Weather asset. |
| getWeatherIndex() | int | O(1) | Returns the cached, integer-based index for the Weather asset. **Warning:** This is only valid after deserialization. |
| getWeight() | double | O(1) | Returns the probability weight for this forecast, used in weighted random selection. |

## Integration Patterns

### Standard Usage
WeatherForecast objects are typically consumed in collections by systems that manage world state, such as a biome or world simulation manager. The manager uses a utility to perform a weighted random selection from the list to determine the next weather state.

```java
// A hypothetical world simulation system
List<WeatherForecast> forecasts = currentBiome.getWeatherForecasts();

// A utility class performs a weighted random selection from the list
WeatherForecast nextForecast = WeightedRandom.select(forecasts);

// The pre-calculated index is used for a fast, direct lookup of the full Weather asset
int weatherIndex = nextForecast.getWeatherIndex();
Weather weatherAsset = Weather.getAssetMap().getByIndex(weatherIndex);

// The simulation then applies the selected weather
world.transitionToWeather(weatherAsset);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new WeatherForecast()`. This bypasses the `CODEC` deserialization pipeline. The resulting object will be in an invalid state, as the critical `processConfig` method will not be called, leaving `weatherIndex` as 0. This can cause the system to silently select the wrong weather type.
- **State Mutation:** Do not modify the fields of a WeatherForecast object after it has been loaded. The server's behavior relies on this configuration being a stable, read-only source of truth.
- **Premature Index Access:** Accessing `getWeatherIndex` on a manually constructed instance is a severe bug. It will return the default `int` value (0), which may correspond to a valid but incorrect Weather asset, leading to difficult-to-trace logical errors in the simulation.

## Data Pipeline
The WeatherForecast object is a product of the server's asset loading pipeline. It represents a transformation of raw configuration data into a runtime-optimized data structure.

> Flow:
> Server Asset File (e.g., biome.json) -> Hytale Codec Deserializer -> **WeatherForecast Instance** (with `processConfig` hook) -> In-Memory Biome Asset -> World Simulation System

---

