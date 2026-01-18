---
description: Architectural reference for NoiseMaskConditionJsonLoader
---

# NoiseMaskConditionJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class NoiseMaskConditionJsonLoader<K extends SeedResource> extends JsonLoader<K, ICoordinateCondition> {
```

## Architecture & Concepts
The NoiseMaskConditionJsonLoader is a specialized deserializer within the procedural generation framework. Its sole responsibility is to translate a specific JSON object schema into a fully realized, in-memory ICoordinateCondition object, specifically a NoiseMaskCondition.

This class is a fundamental component of the engine's **configuration-as-code** pipeline for world generation. It does not contain any generation logic itself; rather, it acts as a factory that constructs the logic objects based on declarative JSON files.

Architecturally, it exemplifies a **compositional pattern**. Instead of containing the logic to parse all sub-components, it delegates the parsing of the noise function and the threshold condition to other, more specialized loaders (NoisePropertyJsonLoader and DoubleConditionJsonLoader, respectively). This creates a decoupled and maintainable system where each loader understands only one specific part of the configuration schema.

## Lifecycle & Ownership
- **Creation:** An instance is created on-demand by a higher-level configuration parser when it encounters a JSON object that represents a noise mask. It is a short-lived, single-purpose object.

- **Scope:** The lifecycle of a NoiseMaskConditionJsonLoader is extremely brief, typically confined to the scope of a single method call within a parent loader. It is instantiated, its `load` method is invoked once, and the instance is then immediately eligible for garbage collection.

- **Destruction:** Managed automatically by the Java Garbage Collector once the reference to the loader instance is no longer reachable. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The loader's state (the source JSON, data folder path, and seed) is provided at construction and is treated as immutable for the object's lifetime. It does not cache results or modify its internal state after initialization.

- **Thread Safety:** This class is **not thread-safe** and is intended for single-threaded use. While its internal state is not modified after construction, it makes no guarantees about the thread safety of the underlying JsonElement or other dependencies.

    **Warning:** Do not share instances of this loader across threads. The expected pattern is for each configuration loading task on a given thread to create its own dedicated loader instances.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ICoordinateCondition | Proportional to JSON complexity | Parses the configured JSON, validates its structure, and returns a new NoiseMaskCondition. Throws IllegalStateException if the required *Threshold* key is missing from the JSON object. |

## Integration Patterns

### Standard Usage
This loader is almost never invoked directly by game systems. It is used internally by other loaders that parse larger, more complex procedural generation assets. The parent loader isolates the relevant JSON snippet and passes it to this class for specialized processing.

```java
// A hypothetical parent loader's logic
JsonElement noiseMaskJson = parentJson.get("myNoiseMask");
SeedString<MySeed> seed = context.getSeed();
Path dataFolder = context.getDataFolder();

// Create a loader for a single, specific task
NoiseMaskConditionJsonLoader<MySeed> loader = new NoiseMaskConditionJsonLoader<>(seed, dataFolder, noiseMaskJson);

// The resulting condition is used by the generation engine
ICoordinateCondition condition = loader.load();
worldGenerator.addCondition(condition);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain an instance of this loader for later use. It is designed to be a single-shot factory. Create a new instance for each distinct JSON object you need to parse.

- **Error Suppression:** The IllegalStateException thrown for a missing *Threshold* key indicates a critical authoring error in the source JSON files. This exception should not be caught and ignored; it must be allowed to propagate up to fail the asset loading process, alerting developers to the malformed data.

## Data Pipeline
The NoiseMaskConditionJsonLoader is a single, focused step in a larger data transformation pipeline that converts static configuration files into live, executable objects for the world generator.

> Flow:
> JSON File on Disk -> Parent Configuration Loader -> JsonElement Snippet -> **NoiseMaskConditionJsonLoader** -> In-Memory NoiseMaskCondition Object -> Procedural Generation Engine

