---
description: Architectural reference for PointEvaluatorJsonLoader
---

# PointEvaluatorJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class PointEvaluatorJsonLoader<T extends SeedResource> extends AbstractCellJitterJsonLoader<T, PointEvaluator> {
```

## Architecture & Concepts
The PointEvaluatorJsonLoader is a specialized factory and deserializer responsible for translating a JSON configuration structure into a concrete PointEvaluator strategy object. It serves as a critical bridge between the declarative world of procedural generation content files and the imperative, executable logic of the generation engine.

Its primary architectural function is to interpret the *intent* of the JSON data. The loader's behavior pivots on the **MeasurementMode** configuration, which acts as a switch to select one of two distinct loading strategies:
1.  **CENTRE_DISTANCE:** Constructs a highly configurable PointEvaluator by composing multiple sub-components, each loaded from a corresponding JSON block (e.g., density, distance range, jitter). This path represents a compositional and flexible approach.
2.  **BORDER_DISTANCE:** Employs a more direct strategy, typically returning a pre-configured singleton instance of BorderPointEvaluator. It may then apply a JitterPointEvaluator using the Decorator pattern if jitter is specified, enhancing the base behavior without altering its core.

This class is a classic example of a transient loader within a larger data-driven system. It is not a service or a long-lived component; it is a short-lived tool for a specific deserialization task.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level configuration orchestrator (e.g., a biome or worldgen loader) when it encounters a JSON object that defines a point evaluation strategy. The caller provides the specific JsonElement sub-tree for this loader to process.
-   **Scope:** Ephemeral. The object's lifetime is strictly confined to a single `load` operation. It holds no state that persists beyond this call and is not registered in any service registry.
-   **Destruction:** The instance is expected to be immediately eligible for garbage collection after the `load` method returns the constructed PointEvaluator. There are no manual cleanup or disposal requirements.

## Internal State & Concurrency
-   **State:** The internal state (seed, dataFolder, json, measurementMode) is provided at construction and is treated as immutable for the object's lifetime. The loader is stateless regarding the loading process itself; it does not cache results or intermediate objects.
-   **Thread Safety:** This class is **not thread-safe** and is designed for thread-confined usage. All internal fields are accessed directly without any synchronization primitives. Sharing an instance of this loader across multiple threads will result in undefined behavior and is strictly prohibited.

## API Surface
The public API is designed for a single, focused operation: loading.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PointEvaluatorJsonLoader(seed, dataFolder, json, ...) | constructor | O(1) | Creates a new loader instance. Multiple overloads exist to provide default values for MeasurementMode. |
| load() | PointEvaluator | O(N) | The primary factory method. Parses the internal JsonElement and constructs the appropriate PointEvaluator object. Complexity is proportional to the number of nodes (N) in the JSON sub-tree. |
| loadPointDistanceFunction() | PointDistanceFunction | O(1) | Public helper to load only the distance function. Exposing this is for potential extensibility, not standard use. |
| loadDensity() | IDoubleCondition | O(M) | Public helper to load only the density condition. Delegates to a sub-loader. |

## Integration Patterns

### Standard Usage
The loader is intended to be used by a parent system that is recursively parsing a larger configuration file. The parent creates the loader, invokes `load`, and consumes the resulting PointEvaluator.

```java
// Hypothetical parent loader processing a larger JSON object
JsonElement pointEvaluatorConfig = rootJsonObject.get("pointEvaluator");

// The loader is created, used once, and then discarded
PointEvaluatorJsonLoader<MySeed> loader = new PointEvaluatorJsonLoader<>(
    currentSeed,
    dataFolderPath,
    pointEvaluatorConfig,
    MeasurementMode.CENTRE_DISTANCE,
    null
);

// The resulting object is the desired product
PointEvaluator evaluator = loader.load();

// The 'evaluator' is now passed to the generation engine
proceduralEngine.setPointEvaluator(evaluator);
```

### Anti-Patterns (Do NOT do this)
-   **Instance Caching or Reuse:** Do not retain an instance of PointEvaluatorJsonLoader for later use. It is a single-use object. Create a new instance for each distinct JSON element you need to parse.
-   **State Mutation:** Do not attempt to modify the loader's state after construction via reflection or other means. Its configuration is intended to be immutable.
-   **Cross-Thread Sharing:** Never pass an instance of this loader to another thread. The lack of synchronization guarantees data races and unpredictable outcomes.

## Data Pipeline
This loader is a specific step in the data transformation pipeline that converts static configuration files into live engine objects.

> Flow:
> Raw JSON File -> GSON Parser -> JsonElement -> **PointEvaluatorJsonLoader** -> PointEvaluator Strategy Object -> Procedural Generation Core

