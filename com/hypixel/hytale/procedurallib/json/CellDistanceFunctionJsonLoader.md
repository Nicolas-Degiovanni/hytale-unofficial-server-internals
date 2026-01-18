---
description: Architectural reference for CellDistanceFunctionJsonLoader
---

# CellDistanceFunctionJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class CellDistanceFunctionJsonLoader<K extends SeedResource> extends JsonLoader<K, CellDistanceFunction> {
```

## Architecture & Concepts
The CellDistanceFunctionJsonLoader is a specialized factory component within the procedural generation framework. Its primary responsibility is to deserialize a JSON configuration object and construct a corresponding CellDistanceFunction instance. This class embodies the data-driven design of the world generation system, allowing complex algorithmic behavior to be defined declaratively in external JSON files.

This loader acts as a bridge between the static data representation (JSON) and the executable logic (the CellDistanceFunction object). It interprets specific keys within the JSON, such as CellType, to select the appropriate distance calculation strategy—for example, choosing between grid-based or hexagonal-based distance functions. It can also compose behaviors by wrapping the base function with decorators, such as the CellBorderDistanceFunction, based on the specified MeasurementMode.

It is a concrete implementation of the abstract JsonLoader, indicating a standardized pattern across the engine for loading game resources from JSON definitions.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor (`new CellDistanceFunctionJsonLoader(...)`) by a higher-level configuration manager or another loader that is processing a larger procedural asset. The caller is responsible for providing the raw JsonElement and other contextual parameters like the seed and data folder path.
- **Scope:** The object's lifetime is extremely short and scoped to a single load operation. It is a classic "builder" or "factory" object that exists only to produce another object.
- **Destruction:** The loader instance becomes eligible for garbage collection immediately after the `load` method returns its result. There is no mechanism or expectation for its reuse.

## Internal State & Concurrency
- **State:** The internal state of the loader is defined by its `final` fields: `seed`, `dataFolder`, `json`, `measurementMode`, and `pointDistanceFunction`. This state is provided at construction and is **immutable** for the lifetime of the object. The class does not cache the result of the `load` operation or any intermediate data.
- **Thread Safety:** This class is **thread-safe**. Its state is immutable, and the `load` method is a pure function with respect to the loader's instance state. It produces a new object based on its initial configuration without causing side effects on itself. Multiple threads can safely call `load` on the same instance, though this is not a typical use case given its transient nature.

## API Surface
The public API is minimal, consisting of the constructors and the primary `load` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CellDistanceFunctionJsonLoader(seed, dataFolder, json, pointDistanceFunction) | constructor | O(1) | Creates a loader with the default CENTRE_DISTANCE measurement mode. |
| CellDistanceFunctionJsonLoader(seed, dataFolder, json, measurementMode, pointDistanceFunction) | constructor | O(1) | Creates a loader with an explicitly defined measurement mode. |
| load() | CellDistanceFunction | O(1) | Constructs and returns the configured CellDistanceFunction. This is the primary entry point. |

## Integration Patterns

### Standard Usage
The loader is intended to be instantiated, used immediately to load the function, and then discarded. It is typically invoked by a service responsible for orchestrating the loading of all procedural generation assets.

```java
// Assume 'configJson' is a JsonElement parsed from a file
// and 'context' provides other necessary parameters.

SeedString<MySeed> seed = ...;
Path dataPath = ...;
PointDistanceFunction pdf = ...;

// 1. Instantiate the loader with the JSON data
CellDistanceFunctionJsonLoader<MySeed> loader = new CellDistanceFunctionJsonLoader<>(
    seed,
    dataPath,
    configJson,
    MeasurementMode.BORDER_DISTANCE,
    pdf
);

// 2. Execute the load operation to get the final object
CellDistanceFunction distanceFunction = loader.load();

// 3. Use the resulting function in the generation algorithm
float distance = distanceFunction.getDistance(cellA, cellB);
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not retain and reuse a loader instance. They are lightweight, carry no cached state, and are designed to be single-use. Create a new loader for each distinct JSON object you need to process.
- **Subclassing for Simple Logic:** Avoid subclassing this loader to add minor variations. The design favors composition and data-driven configuration; new behaviors should be introduced by defining new JSON structures or new, composable CellDistanceFunction implementations.

## Data Pipeline
This loader is a key transformation step in the procedural asset pipeline. It converts declarative data into an executable strategy object.

> Flow:
> JSON File on Disk -> GSON Parser -> `JsonElement` -> **CellDistanceFunctionJsonLoader** -> `CellDistanceFunction` Object -> Procedural Generation Engine

