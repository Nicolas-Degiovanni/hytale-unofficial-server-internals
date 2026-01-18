---
description: Architectural reference for PointGeneratorJsonLoader
---

# PointGeneratorJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class PointGeneratorJsonLoader<K extends SeedResource> extends JsonLoader<K, IPointGenerator> {
```

## Architecture & Concepts
The PointGeneratorJsonLoader is a factory class responsible for translating a declarative JSON configuration into a concrete, executable IPointGenerator object. It serves as a critical bridge between the data-driven procedural generation system and the underlying logic that generates spatial points.

Its core architectural pattern is the **Decorator Pattern**. The loader begins by instantiating a base PointGenerator. It then conditionally wraps this base object in a series of decorators—such as ScaledPointGenerator, DistortedPointGenerator, and OffsetPointGenerator—based on the presence of specific keys within the source JSON. This composition allows for complex and varied point generation behaviors to be defined declaratively without altering engine code.

This class acts as an orchestrator, delegating the construction of its constituent parts (e.g., distance functions, randomizers, evaluators) to other specialized JsonLoader implementations. This separation of concerns keeps the loading logic for each component modular and maintainable.

### Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level configuration parser whenever it encounters a JSON object defining a point generator. It is not a managed service or singleton.
- **Scope:** The object's lifetime is extremely short. It is designed to be single-use, existing only for the duration of the `load` method call.
- **Destruction:** The instance holds no persistent state or external resources. It becomes eligible for garbage collection immediately after the `load` method returns the constructed IPointGenerator.

## Internal State & Concurrency
- **State:** The object's state is effectively immutable. Its core data—the seed, data folder path, and JSON element—are provided during construction and are not modified thereafter. The `load` method is a pure function of this initial state.
- **Thread Safety:** This class is inherently thread-safe due to its immutable state. However, the standard and expected usage pattern is to instantiate a new loader for each distinct loading operation. Sharing a single instance across threads, while safe, is unconventional and not recommended as it provides no performance benefit.

## API Surface
The public contract is minimal, exposing only the primary factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | IPointGenerator | O(D) | Constructs and returns a fully configured IPointGenerator. The complexity is linear in relation to D, the number of decorator keys (e.g., Scale, Offset, Rotate) present in the JSON configuration. |

## Integration Patterns

### Standard Usage
The loader is intended to be instantiated and used immediately by a parent system responsible for parsing procedural generation assets.

```java
// Assume 'worldGenConfig' is a JsonObject and 'context' provides dependencies
JsonElement generatorJson = worldGenConfig.get("myPointGenerator");
SeedString<MySeedResource> seed = context.getSeedString();
Path dataPath = context.getDataFolderPath();

// Instantiate, load, and discard
PointGeneratorJsonLoader<MySeedResource> loader = new PointGeneratorJsonLoader<>(seed, dataPath, generatorJson);
IPointGenerator pointGenerator = loader.load();

// The 'pointGenerator' instance is now a fully composed object ready for use
// The 'loader' instance is no longer needed and can be garbage collected
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not cache or reuse PointGeneratorJsonLoader instances. They are lightweight and tied to a specific JSON element. Caching provides no benefit and can lead to incorrect configurations if the underlying JSON changes.
- **Manual Composition:** Avoid manually instantiating and chaining generator decorators (e.g., `new OffsetPointGenerator(new ScaledPointGenerator(...))`). The loader's purpose is to correctly interpret the JSON and apply decorators in the prescribed order. Bypassing it risks creating invalid or unintended generator behavior.

## Data Pipeline
This class functions as a transformation step, converting declarative data into an executable object.

> Flow:
> JSON Configuration File -> Parent JSON Parser -> **PointGeneratorJsonLoader** -> Composed IPointGenerator Instance -> World Generation System

