---
description: Architectural reference for ValueNoiseJsonLoader
---

# ValueNoiseJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class ValueNoiseJsonLoader<K extends SeedResource> extends JsonLoader<K, ValueNoise> {
```

## Architecture & Concepts
The ValueNoiseJsonLoader is a specialized factory component within the procedural generation framework. Its sole responsibility is to deserialize a specific JSON configuration structure into a fully initialized ValueNoise object. This class acts as a translation layer, converting a declarative data format (JSON) into an executable algorithm component used for world generation.

It extends the generic JsonLoader, signifying its role within a standardized, type-driven deserialization system. This inheritance pattern allows a parent configuration manager to delegate the loading of specific procedural components without needing to understand their internal configuration details. The generic parameter K, constrained to SeedResource, ensures that the resulting noise function is correctly integrated into the engine's deterministic seeding system, allowing for reproducible world generation.

## Lifecycle & Ownership
- **Creation:** An instance is created on-demand by a higher-level configuration parser or factory. This typically occurs when the parser encounters a JSON object that is explicitly typed as a ValueNoise configuration. It is not a persistent service.
- **Scope:** The object's lifetime is exceptionally short. It exists only for the duration of the deserialization task, typically confined to a single method call in the parent loader.
- **Destruction:** The ValueNoiseJsonLoader instance becomes eligible for garbage collection immediately after the `load` method returns its result. It holds no external references and is not registered with any service locator, ensuring a clean and immediate teardown.

## Internal State & Concurrency
- **State:** The internal state of the loader is established at construction and is treated as immutable thereafter. It holds references to the input JSON, a seed, and a data path, but these are used exclusively for read operations during the `load` call. The class performs no internal caching or state modification post-construction.
- **Thread Safety:** This class is not thread-safe and is not intended for concurrent access. It is designed to be instantiated, used once to produce a ValueNoise object, and then discarded. Because it is stateless and transient, it is inherently safe when used as intended within a single-threaded loading process.

    **WARNING:** Sharing a single ValueNoiseJsonLoader instance across multiple threads is an unsupported pattern and will lead to unpredictable behavior.

## API Surface
The public contract is minimal, exposing only the primary factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ValueNoise | O(1) | Constructs and returns a new ValueNoise instance based on the JSON configuration provided at construction time. |

## Integration Patterns

### Standard Usage
This class is an internal component of the configuration system and is not intended for direct use by game logic developers. It is invoked by a parent factory responsible for parsing a larger procedural graph.

```java
// Hypothetical usage within a parent configuration loader
JsonElement noiseConfig = getNoiseConfigFromJson(); // Assume this is the relevant JSON part
SeedString currentSeed = context.getSeed();
Path dataPath = context.getDataPath();

// The system instantiates the loader for a specific, one-off task
ValueNoiseJsonLoader loader = new ValueNoiseJsonLoader(currentSeed, dataPath, noiseConfig);
ValueNoise noiseFunction = loader.load();

// The loader is now discarded, and the resulting 'noiseFunction' is integrated
// into the world generation pipeline.
worldGenerator.setBaseNoise(noiseFunction);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain a reference to a ValueNoiseJsonLoader instance to call `load` multiple times. Each call to `load` would produce an identical object; the loader is not designed for re-configuration or repeated use.
- **Direct Instantiation:** Avoid constructing this class manually in game systems. The procedural library's configuration engine is responsible for selecting the correct loader based on JSON type hints. Bypassing this system can break data-driven behaviors and forward compatibility.

## Data Pipeline
The loader serves as a critical step in the data flow from disk configuration to an in-memory, executable object.

> Flow:
> JSON Configuration File → GSON Parser → JsonElement → **ValueNoiseJsonLoader** → `ValueNoise` Instance → Procedural Generator Engine

