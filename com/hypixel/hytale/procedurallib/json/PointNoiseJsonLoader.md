---
description: Architectural reference for PointNoiseJsonLoader
---

# PointNoiseJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class PointNoiseJsonLoader<K extends SeedResource> extends JsonLoader<K, PointNoise> {
```

## Architecture & Concepts
The PointNoiseJsonLoader is a specialized, single-purpose deserializer within the procedural generation asset pipeline. It acts as a concrete implementation of the generic JsonLoader, responsible for translating a specific JSON object schema into a fully-realized PointNoise domain object.

Its primary architectural function is to decouple the procedural generation engine from the raw data format of its configuration files. By adhering to the JsonLoader contract, it allows a higher-level asset management system to polymorphically load various procedural components without needing to know the specific JSON keys or structure for each one. This class encapsulates the schema knowledge for a point noise source, including required fields (X, Y, Z, InnerRadius, OuterRadius) and their default values.

The generic parameter K, which extends SeedResource, links the loading process to a seeded context, ensuring that any procedural elements derived from this configuration can be generated deterministically.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by a higher-level factory or asset manager when a JSON resource corresponding to a point noise definition is requested. It is not intended for direct instantiation by game logic systems.
- **Scope:** Extremely short-lived and method-scoped. An instance of PointNoiseJsonLoader exists only for the duration of the deserialization task, typically within a single method call of the asset system.
- **Destruction:** The object is immediately eligible for garbage collection once the `load` method returns its PointNoise result. It holds no persistent state and is not retained by any system post-deserialization.

## Internal State & Concurrency
- **State:** The internal state, consisting of the source JsonElement and contextual data like the seed, is provided at construction and is treated as immutable. The class performs a read-only transformation and does not cache results or modify its own state.
- **Thread Safety:** This class is stateless and therefore inherently thread-safe, **provided that the input JsonElement is not mutated by another thread during the `load` operation**. In practice, it is designed for and used within single-threaded asset loading contexts, making concurrent access an edge case. Do not share a single loader instance across multiple threads.

## API Surface
The public contract is minimal, focused exclusively on the transformation operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | PointNoise | O(1) | Deserializes the internal JsonElement into a PointNoise object. Uses predefined default values if keys are missing. Returns null if the source JSON is null. |

## Integration Patterns

### Standard Usage
Correct usage involves obtaining an instance from a managing system and immediately invoking the `load` method to retrieve the domain object. The loader itself is then discarded.

```java
// A hypothetical asset manager is responsible for providing the loader
// with the correct context (seed, path, and json data).
JsonElement noiseConfig = assetManager.getJson("my_world/noise/cave_entrance.json");
SeedString seed = world.getSeed();
Path dataFolder = world.getDataFolderPath();

// The manager constructs the loader for the specific task
PointNoiseJsonLoader loader = new PointNoiseJsonLoader(seed, dataFolder, noiseConfig);

// The loader is used once to produce the result
PointNoise noise = loader.load();

// The 'loader' instance is no longer needed and can be garbage collected.
proceduralSystem.registerNoiseSource(noise);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain and re-use a PointNoiseJsonLoader instance. It is designed as a cheap, transient object. Re-using it provides no performance benefit and violates its intended lifecycle.
- **External Instantiation:** Avoid constructing a PointNoiseJsonLoader manually with `new` in general game logic. The asset pipeline is responsible for its creation to ensure the correct context (seed, data path) is supplied.
- **State Modification:** Do not attempt to modify the underlying JsonElement after passing it to the loader's constructor. The behavior is undefined and may lead to inconsistent state or runtime exceptions.

## Data Pipeline
This class is a discrete step in the data flow from persistent storage to the in-memory procedural engine.

> Flow:
> JSON File on Disk -> Asset System reads `JsonElement` -> **PointNoiseJsonLoader** (Deserialization) -> `PointNoise` Object -> Procedural Generation Engine

---

