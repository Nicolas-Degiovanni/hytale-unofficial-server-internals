---
description: Architectural reference for SimplexNoiseJsonLoader
---

# SimplexNoiseJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class SimplexNoiseJsonLoader<K extends SeedResource> extends JsonLoader<K, NoiseFunction> {
```

## Architecture & Concepts
The SimplexNoiseJsonLoader is a specialized factory component within the procedural generation framework's data-loading pipeline. Its primary architectural function is to act as an adapter between a declarative JSON configuration and the engine's singleton implementation of Simplex noise.

While it conforms to the generic JsonLoader contract, its implementation is a deliberate "no-op" in terms of data parsing. It unconditionally returns the static SimplexNoise.INSTANCE. This design implies that the JSON configuration for Simplex noise serves only as a type specifier (e.g., `"type": "simplex"`) and does not support any further customization. The loader's existence allows the broader procedural system to treat all noise functions polymorphically during the loading phase, even those that are stateless singletons.

**WARNING:** The JsonElement provided to the constructor is completely ignored. This loader does not, and cannot, configure the noise function. It is a provider for a pre-existing, shared instance.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level factory or dispatcher within the procedural asset system. This occurs when the system parses a JSON file and identifies a section that requires a Simplex noise function. It is never created directly by game logic.
- **Scope:** Ephemeral. The object's lifespan is strictly limited to the duration of the `load` method call. It holds no persistent state and is not registered or cached.
- **Destruction:** The object becomes eligible for garbage collection immediately after the `load` method returns the NoiseFunction instance to the calling system.

## Internal State & Concurrency
- **State:** Stateless and immutable. This class contains no mutable fields and its behavior is constant. The state it provides access to—the SimplexNoise singleton—is itself a stateless utility.
- **Thread Safety:** Fully thread-safe. As the object is stateless and the `load` method only returns a static final instance, it can be safely created and used by any thread without synchronization.

## API Surface
The public API consists solely of the `load` method, fulfilling its contract with the parent JsonLoader class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | NoiseFunction | O(1) | Returns the global singleton instance of SimplexNoise. This operation is infallible and has no side effects. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked transparently by the engine's asset loading and procedural generation systems. A developer's only interaction is declarative, via a JSON configuration file.

```json
// Example worldgen.json
{
  "terrainNoise": {
    "type": "simplex" // This declaration triggers the use of SimplexNoiseJsonLoader
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate this class manually in game logic. It is tightly coupled to the JSON loading pipeline and provides no value outside of that context.
- **Expecting Configuration:** Do not attempt to pass JSON with parameters (e.g., frequency, octaves) to this loader. The input JSON is ignored, and such an attempt indicates a misunderstanding of its purpose. For configurable noise, a different loader and noise type must be used.

## Data Pipeline
The SimplexNoiseJsonLoader is a terminal step in a data-loading pipeline, translating a configuration entry into a concrete, reusable service object.

> Flow:
> JSON Configuration File -> Engine JSON Parser -> Loader Factory -> **SimplexNoiseJsonLoader** -> `load()` -> SimplexNoise.INSTANCE -> Procedural Generation System

