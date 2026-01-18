---
description: Architectural reference for CurveNoisePropertyJsonLoader
---

# CurveNoisePropertyJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class CurveNoisePropertyJsonLoader<K extends SeedResource> extends JsonLoader<K, CurveNoiseProperty> {
```

## Architecture & Concepts
The CurveNoisePropertyJsonLoader is a specialized deserializer within the procedural generation asset pipeline. Its sole responsibility is to translate a specific JSON configuration structure into a runtime CurveNoiseProperty object, which is a key component used by the world generation engine to shape noise functions.

Architecturally, this class embodies the **Factory** and **Builder** patterns. It encapsulates the complex logic of parsing a JSON element, resolving dependencies, and constructing a valid domain object.

A critical design feature is its support for a form of dependency injection. The loader can either parse a nested JSON object to create its required NoiseProperty dependency, or it can accept a pre-existing, shared NoiseProperty instance via its constructor. This allows for significant optimization and asset reusability, preventing the same noise configuration from being loaded and stored in memory multiple times.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level asset loading system or another JsonLoader when a JSON object representing a curve noise property is encountered. It is not intended for direct instantiation by game logic systems.
- **Scope:** Extremely short-lived. The loader's lifecycle is bound to a single `load` operation. Once the method returns the constructed CurveNoiseProperty, the loader instance has served its purpose and is eligible for garbage collection.
- **Destruction:** Managed automatically by the Java Garbage Collector. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** The internal state, consisting of the seed, data folder path, JSON element, and an optional noise property, is provided at construction and is effectively immutable. The class does not cache any data and performs no state modification during its lifecycle.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, used, and discarded within the context of a single thread, typically a dedicated asset loading thread. Concurrent calls to `load` on the same instance will lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | CurveNoiseProperty | O(N) | The primary entry point. Deserializes the configured JsonElement into a new CurveNoiseProperty instance. N is the number of nodes in a nested noise JSON. **Warning:** Throws a fatal Error if the "Noise" key is missing and no NoiseProperty was injected at construction, indicating a critical asset configuration failure. |

## Integration Patterns

### Standard Usage
This loader is typically invoked by other parts of the asset loading framework, not by end-user code. The pattern involves passing the relevant JSON sub-tree to the loader to resolve a specific part of a larger configuration.

```java
// Within a parent loader or asset manager
JsonElement curveNoiseJson = parentJson.get("myCurveNoise");
NoiseProperty sharedNoise = getSharedNoiseProperty(); // Assume this is pre-loaded

// Injecting the dependency for optimization
CurveNoisePropertyJsonLoader loader = new CurveNoisePropertyJsonLoader(seed, dataFolder, curveNoiseJson, sharedNoise);
CurveNoiseProperty property = loader.load();

// Use the resulting property in the procedural engine
worldGenerator.applyCurve(property);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not attempt to re-use a single loader instance to parse multiple, different JSON elements. Each loading operation requires a new, dedicated instance.
- **Error Handling:** Do not wrap the `load` call in a generic `try-catch` block expecting to recover. The throwing of an Error, rather than a checked Exception, signals a malformed or incomplete asset file that should fail the entire asset loading process.
- **Direct Instantiation:** Avoid creating this loader with `new` in general game logic. Its creation should be managed by the asset pipeline to ensure the correct seed, data folder, and JSON context are provided.

## Data Pipeline
This class acts as a transformation stage in the asset deserialization pipeline. It consumes a raw data structure and produces a functional engine component.

> Flow:
> JSON Asset File -> Asset System Parser -> JsonElement -> **CurveNoisePropertyJsonLoader** -> CurveNoiseProperty Object -> Procedural Generation Engine

