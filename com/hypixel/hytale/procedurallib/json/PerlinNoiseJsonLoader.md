---
description: Architectural reference for PerlinNoiseJsonLoader
---

# PerlinNoiseJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class PerlinNoiseJsonLoader<K extends SeedResource> extends JsonLoader<K, NoiseFunction> {
```

## Architecture & Concepts
The PerlinNoiseJsonLoader is a specialized factory component within the procedural generation library. Its sole responsibility is to deserialize a specific JSON configuration structure into a fully instantiated and configured PerlinNoise object.

This class embodies the principle of *Configuration as Code*. Instead of hard-coding Perlin noise parameters within the engine, they are defined externally in JSON files. This loader acts as the bridge, translating the declarative JSON data into an executable Java object (a NoiseFunction) that the world generation system can utilize.

It inherits from the generic JsonLoader, providing a concrete implementation for a specific type of noise function. This pattern allows the procedural generation framework to be easily extended with new noise types by simply creating new loader implementations.

## Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level configuration manager or another JsonLoader when it encounters a JSON object designated for Perlin noise generation. The constructor is supplied with the specific JsonElement to parse, a resource seed, and a data folder path.
- **Scope:** The object's lifetime is extremely short and bound to a single loading operation. It is created, its load method is called once, and then it is discarded. It holds no state beyond the scope of this single operation.
- **Destruction:** The instance becomes eligible for garbage collection immediately after the load method returns its result. There are no manual cleanup or disposal mechanisms required.

## Internal State & Concurrency
- **State:** The internal state (the source JsonElement, seed, and data path) is provided at construction and is treated as immutable. The class reads from this state but does not modify it. It does not cache the generated NoiseFunction; each call to load will produce a new instance.
- **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent access. It is intended to be used in a single-threaded context during an asset loading or world generation setup phase. Instantiating and using it across multiple threads is an unsupported pattern that may lead to unpredictable behavior.

## API Surface
The public API is minimal, focusing exclusively on the loading operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | NoiseFunction | O(1) | Parses the internal JSON data and constructs a new PerlinNoise instance. This is the primary entry point. |

## Integration Patterns

### Standard Usage
A developer should almost never interact with this class directly. It is designed to be invoked by an automated procedural generation framework that dispatches loading tasks based on JSON content.

```java
// Hypothetical usage by a parent loader
JsonElement noiseConfig = worldData.get("terrainNoise");
String noiseType = noiseConfig.get("type").getAsString();

NoiseFunction terrainNoise;
if ("Perlin".equals(noiseType)) {
    // The framework instantiates the specific loader for the task
    PerlinNoiseJsonLoader loader = new PerlinNoiseJsonLoader(seed, dataPath, noiseConfig);
    terrainNoise = loader.load();
}

// ... use terrainNoise in world generation
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain an instance of PerlinNoiseJsonLoader after its load method has been called. It is a single-use object.
- **Manual Configuration:** Do not attempt to modify the internal JsonElement after the loader has been constructed. The object expects its initial state to be final.

## Data Pipeline
The class functions as a specific step in the broader configuration-to-object pipeline for procedural generation.

> Flow:
> World Gen JSON File -> GSON Parser -> JsonElement -> **PerlinNoiseJsonLoader** -> PerlinNoise Instance -> World Generator

