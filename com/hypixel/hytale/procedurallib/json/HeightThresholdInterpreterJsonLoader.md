---
description: Architectural reference for HeightThresholdInterpreterJsonLoader
---

# HeightThresholdInterpreterJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class HeightThresholdInterpreterJsonLoader<K extends SeedResource> extends JsonLoader<K, IHeightThresholdInterpreter> {
```

## Architecture & Concepts
The HeightThresholdInterpreterJsonLoader is a specialized factory component within the procedural asset loading pipeline. It does not perform complex parsing itself; instead, it acts as a **polymorphic dispatcher**. Its primary responsibility is to inspect a given JsonElement and delegate the actual instantiation of an IHeightThresholdInterpreter to a more concrete loader class.

This class embodies the **Strategy Pattern**. Based on the structure of the input JSON, it decides whether to use a NoiseHeightThresholdInterpreterJsonLoader for complex, noise-driven thresholds or a BasicHeightThresholdInterpreterJsonLoader for simpler, value-based thresholds.

This architectural choice decouples the consumers of height interpreters (like biome or feature generators) from the specific implementation details of those interpreters. It allows world generation designers to define different kinds of height-based logic within the same configuration structure, and the loading system will transparently select the correct runtime implementation.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level configuration loader when it needs to resolve a height threshold definition from a JSON file. It is not a persistent service.
- **Scope:** The lifetime of a HeightThresholdInterpreterJsonLoader instance is exceptionally short. It is created, its `load` method is called exactly once, and it is then immediately eligible for garbage collection.
- **Destruction:** Managed by the Java Garbage Collector. There are no native resources or manual cleanup steps required.

## Internal State & Concurrency
- **State:** The internal state of this class consists of the `seed`, `dataFolder`, `json`, and `length` provided during construction. This state is effectively immutable as the fields are final or treated as such. The object is stateless with respect to the loading process and performs no caching.
- **Thread Safety:** This class is **not thread-safe** for concurrent use of a single instance. However, it is safe to use within a multi-threaded loading environment provided that each thread creates its own distinct instance. As all internal state is provided at construction and never modified, it is inherently safe in its intended "create, use once, discard" pattern.

## API Surface
The public contract is minimal, focused exclusively on the factory operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | IHeightThresholdInterpreter | O(N) | Inspects the internal JSON and delegates to a concrete loader. The complexity N is determined by the selected downstream loader. |

## Integration Patterns

### Standard Usage
This loader is intended to be used by other, more complex loaders that are parsing larger configuration files for world generation.

```java
// Hypothetical usage within a parent loader
// Assume 'configJson' is a JsonObject for a biome or feature

JsonElement thresholdDef = configJson.get("placementThreshold");
SeedString<BiomeResource> biomeSeed = getBiomeSeed(); // Method to get the current seed context

// Create, use, and discard the loader in one operation
IHeightThresholdInterpreter interpreter = new HeightThresholdInterpreterJsonLoader<>(
    biomeSeed,
    dataFolderPath,
    thresholdDef,
    WORLD_CHUNK_LENGTH
).load();

// The resulting interpreter can now be used by the generation algorithm
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain a reference to an instance of this class to call `load` multiple times. It is designed as a single-use object.
- **State Mutation:** Although not possible with the public API, do not attempt to use reflection to modify the internal `json` or other fields after construction. The dispatch logic depends on this initial state.

## Data Pipeline
This component acts as a routing step in the data flow from raw JSON configuration to a usable in-memory object for the procedural generation engine.

> Flow:
> WorldGen JSON File -> JsonElement for threshold -> **HeightThresholdInterpreterJsonLoader** (Dispatch) -> Concrete Loader (`Noise...` or `Basic...`) -> In-Memory `IHeightThresholdInterpreter` Object

