---
description: Architectural reference for TimeColorAlpha
---

# TimeColorAlpha

**Package:** com.hypixel.hytale.server.core.asset.type.weather.config
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class TimeColorAlpha {
```

## Architecture & Concepts
TimeColorAlpha is a fundamental data structure, not a service or manager. Its sole architectural purpose is to represent a single keyframe in a time-based color gradient. It acts as a data-binding between a specific time of day (represented as a float from 0.0 to 24.0) and a specific RGBA color value.

The most critical architectural feature of this class is the static `CODEC` field. This `BuilderCodec` implementation makes TimeColorAlpha a self-describing entity for Hytale's serialization framework. This design decouples the data (defined in asset files like JSON) from the game logic. Instead of manually parsing configuration files, higher-level systems like a WeatherManager can delegate the entire loading and validation process to the Hytale Codec system, which then populates instances of this class.

An array of TimeColorAlpha objects, deserialized via the static `ARRAY_CODEC`, is the standard mechanism for defining a full 24-hour environmental color cycle for properties such as sky color, fog, or ambient lighting.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale `Codec` system during the server's asset loading phase. A parent system, such as a `WeatherProfileLoader`, initiates the deserialization of a weather configuration asset, which in turn triggers the `CODEC` to instantiate TimeColorAlpha objects. Manual instantiation via `new TimeColorAlpha()` is rare and generally discouraged outside of unit testing.

- **Scope:** The lifetime of a TimeColorAlpha instance is strictly tied to its parent configuration object, for example, a `WeatherProfile`. It is loaded into memory as part of a larger asset and remains for as long as that asset is active. It is not a global or session-scoped object.

- **Destruction:** Instances are marked for garbage collection when their owning configuration object is unloaded or goes out of scope. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The state of a TimeColorAlpha object is **effectively immutable**. While its internal fields are not declared as final, the class exposes no public setters. The design contract is that its state is fully defined at the moment of creation by the constructor or codec and is not intended to change thereafter.

- **Thread Safety:** The class is **thread-safe for read operations**. Due to its effectively immutable nature, multiple threads can safely call `getHour` and `getColor` without external synchronization. The instantiation process via the codec system is assumed to be performed in a single-threaded context per asset load.

## API Surface
The public API is minimal, focusing exclusively on data access and serialization definitions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getHour() | float | O(1) | Returns the hour of the day (0.0-24.0) for this keyframe. |
| getColor() | ColorAlpha | O(1) | Returns the RGBA color associated with this keyframe. |
| CODEC | BuilderCodec | N/A | **Static.** Defines the serialization contract for a single object. |
| ARRAY_CODEC | ArrayCodec | N/A | **Static.** Defines the serialization contract for an array of objects. |

## Integration Patterns

### Standard Usage
TimeColorAlpha is not typically used directly in procedural code. Instead, it is defined declaratively within an asset file (e.g., a JSON file for a weather profile). Game systems then access collections of these objects to interpolate color values for the current game time.

A conceptual asset definition might look like this:
```json
{
  "skyColorGradient": [
    { "Hour": 5.0, "Color": "rgba(45, 60, 90, 255)" },
    { "Hour": 7.0, "Color": "rgba(135, 206, 235, 255)" },
    { "Hour": 18.0, "Color": "rgba(255, 140, 0, 255)" },
    { "Hour": 20.0, "Color": "rgba(25, 25, 112, 255)" }
  ]
}
```

A system would then retrieve the loaded data from a parent configuration object.
```java
// How a system would access the loaded data
WeatherProfile profile = assetManager.load("my_weather.json");
TimeColorAlpha[] skyGradient = profile.getSkyColorGradient();

// Logic would then iterate over the gradient to find the correct color for the current time
ColorAlpha currentColor = WeatherInterpolator.getColorAtTime(skyGradient, world.getTimeOfDay());
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not use reflection or other means to modify the `hour` or `color` fields after an object has been created. This violates its contract as an immutable data carrier and can lead to unpredictable behavior in rendering systems.

- **Misuse as a Dynamic Tracker:** This object defines a static keyframe. It should not be used to hold the *current* state of the world's sky color. Its purpose is to be an element in a larger collection that *defines* the color curve.

## Data Pipeline
TimeColorAlpha serves as a structured data container that bridges raw asset files to in-engine systems.

> Flow:
> Weather Asset File (.json) -> Server Asset Loader -> Hytale Codec Deserializer -> **TimeColorAlpha[]** -> WeatherProfile -> Environmental Rendering System

