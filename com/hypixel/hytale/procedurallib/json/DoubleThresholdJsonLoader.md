---
description: Architectural reference for DoubleThresholdJsonLoader
---

# DoubleThresholdJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class DoubleThresholdJsonLoader<K extends SeedResource> extends JsonLoader<K, IDoubleThreshold> {
```

## Architecture & Concepts
The DoubleThresholdJsonLoader is a specialized deserializer within the procedural generation configuration framework. Its primary function is to translate a flexible JSON data structure into a concrete, type-safe Java representation of a numeric range, represented by the IDoubleThreshold interface.

This class acts as a data-binding component, abstracting the complexities of the configuration format from the core procedural algorithms. The engine can define conditions—such as ore generation or biome placement—using a simple JSON syntax, and this loader ensures it is parsed into a predictable object model. It supports multiple JSON formats for convenience:

*   **Null:** Represents a default condition (either always true or always false).
*   **Single Number:** A shorthand for a range from 0.0 to the specified value.
*   **Two-Element Array:** A standard [min, max] range.
*   **Array of Arrays:** A complex condition representing a union of multiple [min, max] ranges.

By extending the generic JsonLoader, it participates in a larger, convention-based system for loading engine assets and configurations from disk.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level configuration parser or factory when it encounters a JSON field designated as a double threshold. It is not intended for direct creation by end-users.
-   **Scope:** Extremely short-lived. Its existence is typically confined to the scope of a single method call that requires parsing a JsonElement. Once the `load` method returns the IDoubleThreshold object, the loader instance has served its purpose and is eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java garbage collector. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
-   **State:** The internal state (the source JsonElement, data folder path, and default value) is provided during construction and is treated as immutable. The `load` method is a pure transformation of this initial state and does not cause side effects or modify the loader's internal fields.
-   **Thread Safety:** This class is inherently thread-safe for a single operation. Because its state is effectively immutable after construction, the `load` method can be called without external synchronization. However, the class is designed as a transient object, and sharing a single instance across threads is not a standard or recommended use case.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DoubleThresholdJsonLoader(seed, dataFolder, json) | constructor | O(1) | Creates a loader with a default return value of **true** for null JSON input. |
| DoubleThresholdJsonLoader(seed, dataFolder, json, defaultValue) | constructor | O(1) | Creates a loader, specifying the default return value for null JSON input. |
| load() | IDoubleThreshold | O(N) | Parses the internal JsonElement into an IDoubleThreshold object. N is the number of range pairs in the JSON array. Throws IllegalArgumentException on malformed input. |

## Integration Patterns

### Standard Usage
This class is not meant to be invoked directly. It is used by a parent loading system that delegates the parsing of specific JSON fragments.

```java
// Hypothetical usage within a parent configuration loader
JsonElement thresholdJson = biomeConfig.get("heightCondition");
SeedString<?> currentSeed = biomeConfig.getSeed();

// The parent loader creates a transient instance to do the work
DoubleThresholdJsonLoader<SeedResource> loader = new DoubleThresholdJsonLoader<>(currentSeed, dataPath, thresholdJson);
IDoubleThreshold heightCondition = loader.load();

// The procedural engine now uses the resulting object
if (heightCondition.test(currentHeight)) {
    // ...
}
```

### Anti-Patterns (Do NOT do this)
-   **Instance Caching:** Do not retain and reuse instances of DoubleThresholdJsonLoader. They are lightweight and designed to be created on-demand for each parsing operation.
-   **Pre-Processing JSON:** Do not manually parse the JSON and attempt to construct DoubleThreshold objects yourself. The purpose of this loader is to encapsulate the varied and potentially complex parsing logic. Always pass the raw JsonElement to the loader.
-   **Ignoring Exceptions:** The `load` method performs critical validation. Failure to catch its IllegalArgumentException can lead to instability in the procedural generation engine if it receives malformed configuration data.

## Data Pipeline
The loader serves as a specific step in the broader configuration-to-engine pipeline, transforming raw data into a functional system component.

> Flow:
> Biome Configuration File (JSON) -> Engine's GSON Parser -> `JsonElement` representing a threshold -> **DoubleThresholdJsonLoader** -> `IDoubleThreshold` instance -> Procedural Condition Evaluator

