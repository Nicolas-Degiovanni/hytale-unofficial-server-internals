---
description: Architectural reference for DistanceNoiseJsonLoader
---

# DistanceNoiseJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class DistanceNoiseJsonLoader<K extends SeedResource> extends JsonLoader<K, NoiseFunction> {
```

## Architecture & Concepts

The DistanceNoiseJsonLoader is a specialized factory component within the procedural generation framework. Its sole responsibility is to deserialize a specific JSON configuration structure into a fully realized, executable DistanceNoise function. It acts as a concrete implementation of the abstract JsonLoader, bridging declarative world generation data with the engine's imperative code.

Architecturally, this class exemplifies the **Composite Builder** pattern. It does not contain the logic for building all its dependencies from scratch. Instead, it delegates the construction of complex sub-components, such as the CellDistanceFunction and PointEvaluator, to their own dedicated JsonLoader implementations. This approach promotes high cohesion and low coupling, allowing individual noise components to be modified or extended without impacting the parent loader.

A critical design element is the private inner class, LoadedDistanceNoise. This class acts as an **Adapter** that injects a SeedResource into the core DistanceNoise object upon creation. The SeedResource manages thread-local result buffers, which are essential for high-performance, concurrent noise evaluation. By using this adapter, the generic DistanceNoise logic remains decoupled from the engine's specific resource and threading model, while the loader ensures the final object is correctly integrated and optimized for the runtime environment.

## Lifecycle & Ownership

-   **Creation:** An instance of DistanceNoiseJsonLoader is created on-demand by a higher-level procedural graph parser or configuration manager. It is instantiated directly when a JSON object representing a distance noise function needs to be processed. It is not a managed service or singleton.

-   **Scope:** The object's lifetime is exceptionally short and bound to a single operation. It exists only for the duration of the `load` method call.

-   **Destruction:** Once the `load` method returns the constructed NoiseFunction, the DistanceNoiseJsonLoader instance has served its purpose and holds no further state. It becomes immediately eligible for garbage collection. There are no manual cleanup or disposal methods.

## Internal State & Concurrency

-   **State:** The internal state of the loader consists of immutable references to the `seed`, `dataFolder`, and the target `json` element, all provided during construction. This state is read-only and is used exclusively during the `load` operation. The class does not cache any data or maintain mutable state.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed as a transient, single-use object within a single-threaded loading context. The `JsonElement` it operates on is not inherently thread-safe.

    **Warning:** While the loader itself is not thread-safe, the `NoiseFunction` it produces is designed for concurrent evaluation. This is achieved through the injection of the `SeedResource`, which provides thread-local buffers to prevent race conditions during noise generation.

## API Surface

The public contract is minimal, exposing only the primary factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | NoiseFunction | O(N) | Deserializes the JSON configuration into a `DistanceNoise` instance. N is the number of nodes in the JSON sub-tree. Throws exceptions on malformed or missing JSON properties. |

## Integration Patterns

### Standard Usage

This loader is intended to be used by other parts of the configuration loading system. A developer should never need to instantiate it directly unless extending the core procedural loading mechanism.

```java
// Hypothetical usage within a parent loader
JsonElement config = parseSomeFile();
SeedString<MySeed> seed = getSeedForContext();
Path dataRoot = getDataFolderPath();

// The loader is created, used, and immediately discarded
DistanceNoiseJsonLoader<MySeed> loader = new DistanceNoiseJsonLoader<>(seed, dataRoot, config);
NoiseFunction distanceNoise = loader.load();

// The resulting function is now ready for use in the engine
proceduralGraph.addNode(distanceNoise);
```

### Anti-Patterns (Do NOT do this)

-   **Instance Reuse:** Do not retain a reference to a DistanceNoiseJsonLoader instance for later use. Each load operation for a distinct JSON object requires a new loader instance.

-   **Concurrent Loading:** Do not call the `load` method from multiple threads on the same instance. This will lead to unpredictable behavior and potential crashes. The loading process is designed to be a single-threaded bootstrap phase.

-   **State Modification:** Do not attempt to modify the internal state of the loader via reflection after construction. The object assumes its initial context is immutable.

## Data Pipeline

The DistanceNoiseJsonLoader is a key transformation step in the procedural content pipeline, converting static data into executable logic.

> Flow:
> Raw JSON File -> Engine Configuration Parser -> `JsonElement` -> **DistanceNoiseJsonLoader** -> `NoiseFunction` (in-memory object) -> Procedural World Generator

