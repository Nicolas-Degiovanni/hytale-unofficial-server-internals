---
description: Architectural reference for GridNoiseJsonLoader
---

# GridNoiseJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class GridNoiseJsonLoader<K extends SeedResource> extends JsonLoader<K, NoiseFunction> {
```

## Architecture & Concepts
The GridNoiseJsonLoader is a specialized deserializer within the procedural generation library. Its sole responsibility is to translate a specific JSON object structure into a fully configured and operational GridNoise instance.

This class acts as a concrete implementation of a factory pattern, inheriting from the generic JsonLoader. This design indicates a broader framework where various components of the procedural engine (e.g., noise functions, biome definitions, object placement rules) are loaded from data files using a consistent, type-safe mechanism. GridNoiseJsonLoader encapsulates the specific business logic for parsing grid noise parameters, including handling default values and validating required fields.

By isolating this logic, the core procedural engine remains decoupled from the specifics of the JSON configuration format, allowing the data schema to evolve without impacting the mathematical components of the noise generation system.

## Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level configuration manager or factory when it encounters a JSON block designated for grid noise. The caller must provide the specific JsonElement representing the grid noise configuration, along with a context-aware seed and data folder path.
- **Scope:** The object's lifecycle is ephemeral and strictly bound to a single `load` operation. It is a single-use tool.
- **Destruction:** The instance holds no persistent resources and is eligible for garbage collection immediately after the `load` method returns its result. The created NoiseFunction object is what persists, not the loader itself.

## Internal State & Concurrency
- **State:** The GridNoiseJsonLoader is stateful, but its state is immutable after construction. It holds references to the input JsonElement, seed, and data path, which are stored in the parent JsonLoader class. This state is read-only during the `load` operation. The class performs no caching.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded execution within a larger data loading pipeline. Accessing a single instance from multiple threads will lead to undefined behavior.

## API Surface
The public contract is minimal, consisting of the constructor and the primary `load` method. Protected methods are for internal implementation or extension and are not considered part of the stable API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | NoiseFunction | O(1) | Parses the internal JSON data and constructs a new GridNoise object. Throws an Error if a required thickness value is missing and no default is provided. |

## Integration Patterns

### Standard Usage
The loader is intended to be instantiated, used once to produce a NoiseFunction, and then discarded. It is a transient object in the data loading pipeline.

```java
// Assume 'configElement' is a JsonElement from a larger config file.
// Assume 'contextSeed' and 'dataRoot' are provided by the loading environment.

GridNoiseJsonLoader<MySeedType> loader = new GridNoiseJsonLoader<>(
    contextSeed,
    dataRoot,
    configElement
);

NoiseFunction gridNoise = loader.load();

// The 'gridNoise' object is now ready for use in the world generator.
// The 'loader' instance is no longer needed.
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain an instance of GridNoiseJsonLoader for later use. Each loading operation for a distinct JSON configuration must use a new instance.

- **External State Mutation:** Do not modify the JsonElement passed to the constructor from another thread while the `load` method is executing. This will cause a severe race condition.

- **Direct Instantiation without a Framework:** While possible, creating a GridNoiseJsonLoader manually is discouraged. It is designed to be invoked by a parent system that walks a larger configuration tree and dispatches to the correct loader.

## Data Pipeline
The GridNoiseJsonLoader is a critical transformation step in the procedural content generation pipeline. It converts declarative data (JSON) into executable logic (a NoiseFunction object).

> Flow:
> World Configuration File (.json) -> GSON Parser -> JsonElement -> **GridNoiseJsonLoader** -> GridNoise Object -> World Generation Engine

---

