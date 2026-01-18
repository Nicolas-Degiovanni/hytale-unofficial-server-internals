---
description: Architectural reference for BlendNoisePropertyJsonLoader
---

# BlendNoisePropertyJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class BlendNoisePropertyJsonLoader<K extends SeedResource> extends JsonLoader<K, BlendNoiseProperty> {
```

## Architecture & Concepts
The BlendNoisePropertyJsonLoader is a specialized deserializer within the procedural generation framework. Its sole responsibility is to translate a specific JSON object structure into a fully configured, in-memory BlendNoiseProperty instance. This class is a critical link in the chain that transforms declarative world generation configuration files into executable logic.

It operates as a composite loader. While it manages the overall structure of a "BlendNoise" JSON block, it delegates the complex task of parsing nested noise definitions to the NoisePropertyJsonLoader. This follows a principle of separation of concerns, allowing each loader to be an expert in its specific domain. The loader's primary function is to orchestrate the parsing of three key components:
1.  An **Alpha** noise property, which acts as a controller.
2.  An array of **Noise** properties, which are the different noises to be blended.
3.  An array of **Thresholds**, which define the blend points controlled by the Alpha noise.

After deserialization, it performs a crucial validation step to ensure the structural integrity of the configuration, enforcing that the number of noises matches the number of thresholds and that thresholds are in strictly ascending order.

## Lifecycle & Ownership
-   **Creation:** An instance is created on-demand by a higher-level configuration parser or factory. It is instantiated with the specific JsonElement it is expected to parse, along with contextual dependencies like the SeedResource and the root data folder path for resolving any further linked resources.
-   **Scope:** The object's lifetime is ephemeral. It is designed to be single-use and exists only for the duration of a single `load` operation.
-   **Destruction:** It holds no external references and is not registered with any service locator. It becomes eligible for garbage collection immediately after the `load` method returns its result.

## Internal State & Concurrency
-   **State:** The loader is stateful, containing references to the seed, data folder, and the target JSON element. This state is provided at construction and is treated as immutable for the object's short lifetime. It performs no internal caching.
-   **Thread Safety:** This class is **not thread-safe**. It is designed for synchronous, single-threaded execution. The `load` method and its internal helpers depend on the instance-specific state. Instantiating and using a single loader across multiple threads will lead to undefined behavior. However, creating separate instances of BlendNoisePropertyJsonLoader on different threads is safe as they do not share any state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | BlendNoiseProperty | O(N) | Deserializes the JSON element provided at construction into a BlendNoiseProperty. N is the number of noises in the configuration. Throws IllegalStateException if the number of noises and thresholds do not match or if thresholds are not in ascending order. |

## Integration Patterns

### Standard Usage
This loader is not intended to be used directly by game logic but rather by other parts of the configuration loading system. A parent loader would identify a "BlendNoise" JSON object and delegate its parsing to this class.

```java
// Assume 'parentJson' is a JsonObject from a larger configuration file
// and 'contextSeed' and 'contextDataFolder' are available.

JsonElement blendNoiseJson = parentJson.get("myBlendNoise");

BlendNoisePropertyJsonLoader<MySeed> loader = new BlendNoisePropertyJsonLoader<>(
    contextSeed,
    contextDataFolder,
    blendNoiseJson
);

// The resulting property is now ready for use in the procedural engine
BlendNoiseProperty property = loader.load();
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not retain an instance of this loader for later use. It is bound to the specific JsonElement passed during construction and is not designed to be reconfigured or reused.
-   **Ignoring Exceptions:** The `load` method throws an unchecked IllegalStateException on validation failure. This indicates a critical error in the source JSON configuration. This exception must be caught by the higher-level loading system to prevent a corrupt or unstable state in the procedural generation engine.

## Data Pipeline
The class functions as a single step in a larger data transformation pipeline that converts on-disk assets into live game objects.

> Flow:
> JSON File on Disk -> GSON Parser -> `JsonElement` -> **BlendNoisePropertyJsonLoader** -> `BlendNoiseProperty` Object -> Procedural Generation Engine

