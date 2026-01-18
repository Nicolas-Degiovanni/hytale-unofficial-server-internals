---
description: Architectural reference for OldSimplexNoiseJsonLoader
---

# OldSimplexNoiseJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class OldSimplexNoiseJsonLoader<K extends SeedResource> extends JsonLoader<K, NoiseFunction> {
```

## Architecture & Concepts
The OldSimplexNoiseJsonLoader is a specialized factory component within the procedural generation asset pipeline. Its sole responsibility is to act as a bridge between a JSON configuration file and the engine's singleton instance of the legacy Simplex Noise implementation.

Architecturally, this class represents a "static mapping" loader. When the procedural world generator parses a configuration that specifies the use of `OldSimplexNoise`, it dispatches the loading request to an instance of this class. The key design characteristic is that this loader **completely ignores the content** of the provided JsonElement. Its existence implies that the *type* of the resource being requested is sufficient to resolve the dependency.

This pattern is used to integrate singleton or globally-defined systems into a data-driven, polymorphic loading framework. It allows configuration files to reference a shared, immutable resource without needing to define its parameters, as those parameters are globally fixed.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level factory or registry, such as a `ProceduralContentFactory`, during the parsing of world generation assets. It is never created directly by game logic.
- **Scope:** Extremely short-lived and method-scoped. An instance of this loader exists only for the duration of a single `load` operation.
- **Destruction:** The object is immediately eligible for garbage collection after the `load` method returns. No external systems should ever retain a reference to an OldSimplexNoiseJsonLoader instance.

## Internal State & Concurrency
- **State:** This object is effectively **stateless**. While it receives a seed, data folder, and JSON element in its constructor, these are passed to the superclass and are not used by this implementation's `load` method. It holds no internal mutable state.
- **Thread Safety:** The `load` operation is unconditionally thread-safe. It performs a single, non-blocking read of a static final field (`OldSimplexNoise.INSTANCE`). Multiple threads can create and use separate instances of this loader concurrently without any risk of race conditions or data corruption. The safety of the returned `NoiseFunction` object is a separate concern, though the target `OldSimplexNoise` singleton is designed to be thread-safe.

## API Surface
The public contract is minimal, consisting only of the inherited `load` method. The constructor is public for framework access but is not considered part of the primary API for developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | NoiseFunction | O(1) | Returns the global singleton instance of OldSimplexNoise. This operation will never fail and does not throw exceptions. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use. It is invoked polymorphically by the procedural asset system. A developer would typically define a noise function in a JSON file, and the engine's loading mechanism would select this loader automatically.

```java
// Hypothetical usage within a central factory
// Developer does NOT write this code; this is how the engine uses the loader.

JsonElement config = parseJsonFile("worldgen/noise/legacy_terrain.json");
String noiseType = config.getAsJsonObject().get("type").getAsString(); // "OldSimplexNoise"

// The factory maps the string type to the correct loader class
JsonLoader<?, NoiseFunction> loader = factory.createLoaderForType(noiseType, seed, dataFolder, config);

// loader is now an instance of OldSimplexNoiseJsonLoader
NoiseFunction noise = loader.load(); // Returns OldSimplexNoise.INSTANCE
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new OldSimplexNoiseJsonLoader()`. The procedural framework is responsible for creating loaders based on JSON definitions. Direct creation bypasses the asset management system.
- **Expecting Parameterization:** Do not attempt to pass parameters via the JSON object. This loader is designed to ignore them, and any configuration provided will be discarded. This behavior is intentional.
- **Caching the Loader:** Do not store and reuse an instance of this loader. It is a cheap, transient object. Cache the resulting `NoiseFunction` if necessary, not the factory that creates it.

## Data Pipeline
The flow of data through this component is a simple resolution step, transforming a configuration reference into a concrete object reference.

> Flow:
> World Generation Asset (JSON) -> Asset Parser -> **OldSimplexNoiseJsonLoader** (Instantiation) -> `load()` call -> `OldSimplexNoise.INSTANCE` -> Procedural Generator Engine

