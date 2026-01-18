---
description: Architectural reference for GradientNoisePropertyJsonLoader
---

# GradientNoisePropertyJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class GradientNoisePropertyJsonLoader<K extends SeedResource> extends JsonLoader<K, GradientNoiseProperty> {
```

## Architecture & Concepts
The GradientNoisePropertyJsonLoader is a specialized component within the procedural generation framework's configuration system. It operates as a concrete implementation of the abstract JsonLoader, tasked with a single, focused responsibility: parsing a specific JSON structure and materializing it into a GradientNoiseProperty object.

This class acts as a deserializer that builds upon an existing noise configuration. It does not create a noise function from scratch; instead, it reads parameters that *modify* a pre-existing NoiseProperty. This design pattern decouples the definition of base noise from the application of gradient-based transformations, allowing for modular and reusable world generation configurations. It is a key part of the data-driven pipeline that translates human-readable JSON files into executable procedural generation logic.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level configuration manager or factory. The creator is responsible for first parsing a base NoiseProperty and then providing it, along with the relevant JsonElement, to this loader's constructor. It is never created in isolation.
- **Scope:** The object's lifetime is exceptionally short. It is designed to be single-use, existing only for the duration of the `load` method call. Once the GradientNoiseProperty is returned, the loader instance has served its purpose and is immediately eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency
- **State:** The internal state consists of the constructor-injected dependencies: the source JsonElement and the base NoiseProperty. This state is established at creation and is not modified thereafter, making the object effectively immutable from a behavioral standpoint. It is a stateful container for a single, transient operation.
- **Thread Safety:** This class is not thread-safe for concurrent modification, but it is safe for its intended use. It is designed to be instantiated and used by a single thread within a larger, single-threaded loading process. Sharing an instance of this loader across multiple threads is an unsupported and dangerous pattern. No internal locking mechanisms are present.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | GradientNoiseProperty | O(1) | Parses the internal JsonElement and constructs a new GradientNoiseProperty. This is the primary entry point and purpose of the class. |

## Integration Patterns

### Standard Usage
This loader is intended to be used by a parent system that orchestrates the loading of complex procedural assets. The parent is responsible for providing the correct context.

```java
// Assume 'parentJson' is a larger configuration object
// and 'baseNoise' has already been loaded.

JsonElement gradientJson = parentJson.get("gradientSettings");
NoiseProperty baseNoise = someFactory.loadNoise(parentJson.get("baseNoise"));

// The loader is created with all necessary context
GradientNoisePropertyJsonLoader loader = new GradientNoisePropertyJsonLoader(seed, dataFolder, gradientJson, baseNoise);

// The loader's single purpose is to produce the property object
GradientNoiseProperty finalProperty = loader.load();
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not hold a reference to this loader and attempt to use it to load a second, different JSON object. It is intrinsically tied to the JsonElement provided at construction.
- **Incomplete Context:** Instantiating this class with a null NoiseProperty or JsonElement will result in a NullPointerException during the `load` call. The loader's contract requires valid, non-null dependencies.

## Data Pipeline
This class is a transformation stage in the configuration loading pipeline. It converts structured text data into a functional in-memory object used by the procedural engine.

> Flow:
> Raw JSON File -> GSON Parser -> `JsonElement` -> **GradientNoisePropertyJsonLoader** -> `GradientNoiseProperty` Instance -> World Generation System

