---
description: Architectural reference for NoiseFunctionJsonLoader
---

# NoiseFunctionJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class NoiseFunctionJsonLoader<K extends SeedResource> extends JsonLoader<K, NoiseFunction> {
```

## Architecture & Concepts
The NoiseFunctionJsonLoader serves as a high-level **dispatcher** within the procedural asset loading framework. It is not a loader in the traditional sense; it does not contain the logic to parse any specific noise algorithm. Instead, its sole responsibility is to inspect a given JSON payload, identify the required noise algorithm via the **NoiseType** key, and delegate the final parsing and instantiation to a specialized, concrete loader.

This class embodies the **Factory** and **Strategy** patterns. It decouples the generic configuration loading process from the specific implementations of various noise functions (e.g., Perlin, Simplex, Cellular). This design is critical for engine extensibility, as it allows developers to introduce new noise algorithms by simply:
1.  Adding a new entry to the NoiseTypeJson enum.
2.  Implementing a corresponding JsonLoader for the new algorithm.

No modifications are required to this dispatcher class, adhering to the open/closed principle. It acts as a stable, central routing point for hydrating NoiseFunction objects from their JSON definitions.

### Lifecycle & Ownership
-   **Creation:** An instance is created on-demand by a higher-level configuration or asset parser whenever it encounters a JSON object that defines a noise function. It is not managed by a dependency injection container or service registry.
-   **Scope:** The object's lifetime is extremely short and is scoped to the execution of a single `load` operation. It is a classic transient object, created, used once, and then immediately discarded.
-   **Destruction:** The instance becomes eligible for garbage collection as soon as the `load` method returns its `NoiseFunction` result. It holds no persistent state and is not registered in any global context.

## Internal State & Concurrency
-   **State:** The class holds immutable references to the `seed`, `dataFolder`, and `json` element provided during construction. This state is used exclusively for the duration of the `load` call and is passed through to the delegated loader. It performs no internal caching.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be instantiated and used within the context of a single-threaded asset loading operation. Parallelizing noise function loading should be achieved by creating a separate NoiseFunctionJsonLoader instance for each task on each thread.

## API Surface
The public contract is minimal, consisting of the constructor for instantiation and the `load` method to execute the operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | NoiseFunction | O(1) | Executes the dispatch logic. Throws IllegalStateException if the NoiseType key is absent from the JSON payload. The overall complexity is determined by the delegated loader. |

## Integration Patterns

### Standard Usage
This class is intended to be used by other parts of the asset loading system. A developer would typically obtain a JsonElement from a larger configuration file and then instantiate this loader to process that specific segment.

```java
// Assume 'configElement' is a JsonElement representing a noise function
// and 'parentSeed' is the SeedString from the parent loader.

SeedString<MySeed> seed = parentSeed.append("worldgen.caves");
Path dataPath = Paths.get("./assets");

// Instantiate the loader for a specific, single-use operation
NoiseFunctionJsonLoader<MySeed> loader = new NoiseFunctionJsonLoader<>(seed, dataPath, configElement);

// Hydrate the NoiseFunction object
NoiseFunction caveNoise = loader.load();

// The 'loader' instance should now be discarded.
```

### Anti-Patterns (Do NOT do this)
-   **Instance Reuse:** Do not retain a reference to this loader for later use. It is stateful for a single operation and must be re-instantiated with new context for each JSON object you need to parse.
-   **External State Mutation:** Do not modify the JsonElement passed to the constructor from another thread while the `load` method is executing. This will lead to unpredictable behavior and likely cause a runtime crash.
-   **Ignoring Exceptions:** The IllegalStateException thrown for a missing NoiseType is a critical asset validation failure. It must not be caught and ignored. This indicates a malformed asset that will prevent procedural generation from functioning correctly.

## Data Pipeline
The NoiseFunctionJsonLoader is a key transformation step in the procedural asset pipeline. It acts as a router, directing the data flow to the correct specialized parser based on the data's content.

> Flow:
> JSON Asset on Disk -> GSON Parser -> JsonElement -> **NoiseFunctionJsonLoader** (Inspects "NoiseType") -> Specialized Loader (e.g., PerlinNoiseJsonLoader) -> In-Memory NoiseFunction Instance

