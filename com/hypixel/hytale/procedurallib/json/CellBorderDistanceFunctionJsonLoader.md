---
description: Architectural reference for CellBorderDistanceFunctionJsonLoader
---

# CellBorderDistanceFunctionJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class CellBorderDistanceFunctionJsonLoader<K extends SeedResource> extends JsonLoader<K, BorderDistanceFunction> {
```

## Architecture & Concepts
The CellBorderDistanceFunctionJsonLoader is a specialized factory component within the Procedural Generation library's configuration system. Its sole responsibility is to deserialize a specific `JsonElement` into a fully configured `BorderDistanceFunction` object.

This class embodies the **Composition over Inheritance** principle for complex data parsing. Instead of containing all the logic to parse every part of its associated JSON structure, it delegates the parsing of subordinate components—specifically the `PointEvaluator` and density condition—to a more specialized `PointEvaluatorJsonLoader`. This design makes the loading system modular and extensible; changes to the `PointEvaluator` JSON format only require modifications to its dedicated loader, isolating the impact.

It functions as a transient, single-use builder that translates a declarative JSON configuration into an executable Java object used by the procedural engine.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level JSON parsing authority when it encounters a JSON object designated as a border distance function. It is never managed by a dependency injection container or service registry.
- **Scope:** The object's lifetime is exceptionally brief, typically confined to the execution of a single method in its parent loader. It is created, its `load` method is called once, and it is then immediately discarded.
- **Destruction:** As a short-lived, stack-allocated object with no external references, it becomes eligible for garbage collection as soon as the `load` method returns its result.

## Internal State & Concurrency
- **State:** The internal state of the loader (`seed`, `dataFolder`, `json`, `distanceFunction`) is provided entirely through its constructor and is treated as immutable. The class does not modify its own state during its lifecycle.
- **Thread Safety:** This class is inherently thread-safe. It performs read-only operations on its initial state and produces a new, distinct object on each invocation of `load`. Multiple instances can be safely used across different threads to parse different JSON elements concurrently, making it suitable for parallelized asset loading pipelines.

## API Surface
The public contract is minimal, exposing only the primary factory method. Protected methods are intended for specialization by subclasses and are not part of the public API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | BorderDistanceFunction | O(N) | Constructs and returns a new `BorderDistanceFunction`. The complexity N is proportional to the size and depth of the underlying JSON structure being parsed by its delegate loaders. |

## Integration Patterns

### Standard Usage
This class is an internal framework component and is not intended for direct use in gameplay logic. It is invoked by other loaders as part of a larger configuration deserialization process.

```java
// Hypothetical invocation from a parent configuration loader
JsonElement borderFuncJson = rootConfig.get("borderFunction");
CellDistanceFunction dependency = context.getDistanceFunction(); // Assumes dependency is already loaded

// The loader is created, used, and immediately discarded
CellBorderDistanceFunctionJsonLoader<MySeed> loader = new CellBorderDistanceFunctionJsonLoader<>(seed, dataPath, borderFuncJson, dependency);
BorderDistanceFunction function = loader.load();

// The resulting 'function' is now passed to the procedural generation engine
proceduralEngine.registerFunction(function);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain an instance of this loader to call `load` multiple times. It is designed to be a single-use object for a specific `JsonElement`.
- **External Instantiation:** Application-level code should never instantiate this loader directly. The procedural generation framework's configuration system is solely responsible for its creation and invocation.

## Data Pipeline
This loader acts as a transformation stage in the broader configuration-to-engine pipeline. It consumes a raw data structure and produces a functional, in-memory object.

> Flow:
> Raw JSON File -> GSON Parser -> `JsonElement` -> **CellBorderDistanceFunctionJsonLoader** -> `BorderDistanceFunction` -> Procedural Generation Engine

