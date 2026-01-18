---
description: Architectural reference for TimeFloat
---

# TimeFloat

**Package:** com.hypixel.hytale.server.core.asset.type.weather.config
**Type:** Data Structure / DTO

## Definition
```java
// Signature
public class TimeFloat {
```

## Architecture & Concepts
TimeFloat is a fundamental data structure used throughout the server's configuration-driven systems, particularly the weather engine. It is not a service or manager, but rather a simple value object that represents a single, discrete point on a time-series graph.

Its primary role is to map a specific hour of the in-game day (from 0.0 to 24.0) to a corresponding floating-point value. This allows designers and developers to define curves for various environmental parameters, such as temperature, fog density, or light level, which change over the course of a day.

The most critical architectural feature of this class is the static **CODEC** field. This self-describing serialization contract allows the Hytale asset pipeline to automatically instantiate and populate TimeFloat objects from configuration files (e.g., JSON). This pattern decouples the game logic from the raw configuration data, enabling data-driven design.

## Lifecycle & Ownership
- **Creation:** TimeFloat instances are almost exclusively created by the Hytale **Codec** system during asset loading. When a larger configuration asset, such as a WeatherProfile, is deserialized from disk, its internal list of time-value pairs is parsed, and the `TimeFloat.CODEC` is invoked to construct each instance. Manual instantiation is rare and typically only done for unit testing or procedural generation.
- **Scope:** The lifetime of a TimeFloat object is strictly bound to its parent configuration object. It is a transient, short-lived object.
- **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for cleanup as soon as their containing configuration asset is unloaded or goes out of scope. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The class holds two float values: *hour* and *value*. While the fields are technically mutable (marked as protected), the public API exposes no setters. This makes the object **effectively immutable** after its initial construction by the codec. All state is contained within the instance.
- **Thread Safety:** Due to its effective immutability, TimeFloat is inherently **thread-safe for reads**. Multiple threads can safely access the `getHour` and `getValue` methods without synchronization. The creation process via the asset pipeline is assumed to be handled in a single-threaded or thread-contained manner.

## API Surface
The public contract is minimal, focusing on data access and serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | N/A | **Critical.** Static field defining serialization and validation rules. Used by the asset system. |
| getHour() | float | O(1) | Returns the time component of the data point (0.0-24.0). |
| getValue() | float | O(1) | Returns the value component associated with the hour. |

## Integration Patterns

### Standard Usage
TimeFloat is not intended to be used directly in most game logic. Instead, systems consume collections of TimeFloat objects, typically loaded from an asset, to interpolate values for the current in-game time.

```java
// Example: A weather system retrieving a value for the current time.
// Assume 'weatherProfile.getTemperatureCurve()' returns a List<TimeFloat> loaded from a config file.

List<TimeFloat> temperatureCurve = weatherProfile.getTemperatureCurve();
float currentTime = world.getTimeOfDay(); // e.g., 13.5

// Logic would then interpolate between the two TimeFloat points surrounding the current time.
float currentTemp = WeatherInterpolator.getValueFromCurve(temperatureCurve, currentTime);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid `new TimeFloat(h, v)` in general game logic. Configuration should be defined in data files and loaded via the asset system. Direct instantiation couples your logic to specific magic numbers.
- **State Modification:** Do not create subclasses of TimeFloat to modify its state after construction. The system relies on these objects being immutable once loaded.
- **Misuse as a Generic Pair:** This class is specifically for time-series data. Do not use it as a generic key-value pair for non-time-related data; use a more appropriate data structure.

## Data Pipeline
TimeFloat serves as a structured representation of raw configuration data as it flows from disk into the game engine.

> Flow:
> WeatherProfile.json -> Asset Loading Service -> **TimeFloat.CODEC** -> **TimeFloat Instance** (in memory) -> Weather Simulation Engine

