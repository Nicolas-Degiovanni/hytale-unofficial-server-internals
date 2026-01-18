---
description: Architectural reference for TimeColor
---

# TimeColor

**Package:** com.hypixel.hytale.server.core.asset.type.weather.config
**Type:** Data Structure

## Definition
```java
// Signature
public class TimeColor {
```

## Architecture & Concepts
TimeColor is a fundamental data structure that represents a single keyframe in a time-based color gradient. Its primary role is to associate a specific time of day, represented as a floating-point hour, with a corresponding Color value.

This class is not intended to be a standalone entity. It is almost exclusively used within collections to define the color transitions for environmental effects like sky, fog, or ambient light throughout the Hytale day-night cycle.

The most critical architectural aspect of TimeColor is its deep integration with the Hytale serialization framework via the static CODEC and ARRAY_CODEC fields. These `Codec` instances define the contract for how TimeColor objects are parsed from and written to asset configuration files, making it a data-driven component. The engine's asset loading system relies on these codecs to hydrate weather profiles from disk into in-memory object graphs.

## Lifecycle & Ownership
- **Creation:** TimeColor instances are created in two primary scenarios:
    1.  By the Hytale `Codec` system during the deserialization of weather configuration assets. The `BuilderCodec` uses the protected no-argument constructor and populates the fields from the source data.
    2.  Programmatically via the public constructor `new TimeColor(float, Color)`, typically for dynamic or procedurally generated weather effects.
- **Scope:** This is a transient, short-lived object. Its lifetime is strictly bound to the parent configuration object that contains it, such as a `WeatherProfile`. It does not persist beyond the lifetime of its owner.
- **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for collection as soon as their parent configuration object is unloaded or goes out of scope. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The internal state, consisting of `hour` and `color`, is mutable. Although the fields are protected, they are directly assigned during the codec-driven creation process. Once an instance is fully constructed and part of a loaded asset, it should be treated as immutable.

- **Thread Safety:** This class is **not thread-safe**. It contains no synchronization primitives.
    **WARNING:** Concurrent modification of a TimeColor instance will lead to undefined behavior. Configuration data, including TimeColor objects, is typically loaded on a single thread and subsequently treated as read-only by the rest of the engine. Do not share and modify instances across threads without external locking.

## API Surface
The public API is minimal, focusing on data access and serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec<TimeColor> | N/A | Static codec for serializing and deserializing a single TimeColor object. |
| ARRAY_CODEC | ArrayCodec<TimeColor> | N/A | Static codec for serializing and deserializing an array of TimeColor objects. |
| getHour() | float | O(1) | Returns the time-of-day keyframe, from 0.0 to 24.0. |
| getColor() | Color | O(1) | Returns the Color value associated with the keyframe. |

## Integration Patterns

### Standard Usage
Developers typically define TimeColor objects declaratively within a larger asset file (e.g., JSON). The engine's asset system uses the provided codecs to load this data. The primary consumer is the weather or lighting system, which interpolates between the colors in a list of TimeColor objects based on the current game time.

```java
// Engine-level code (conceptual)
// A WeatherProfile would contain a list of TimeColor objects loaded via ARRAY_CODEC.
WeatherProfile profile = assetManager.load("weather/default.json");

// The rendering system gets the color for the current time.
float currentTime = world.getTimeOfDay(); // e.g., 13.5f
Color skyColor = profile.getSkyColorGradient().getColorAt(currentTime);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not modify a TimeColor object after it has been loaded as part of a configuration. This violates the principle of immutable configuration data and can lead to unpredictable visual artifacts or state corruption if the configuration is shared.
- **Direct Instantiation for Configuration:** Avoid creating TimeColor instances manually in code to build static weather profiles. This makes the configuration rigid and difficult to change. Data should live in asset files.

## Data Pipeline
TimeColor serves as the in-memory representation of configuration data defined on disk.

> Flow:
> Weather Asset File (e.g., JSON) -> Asset Loading Service -> **TimeColor.ARRAY_CODEC** -> Array of **TimeColor** instances -> Weather System -> Color Interpolation Logic -> Final Rendered Scene Color

