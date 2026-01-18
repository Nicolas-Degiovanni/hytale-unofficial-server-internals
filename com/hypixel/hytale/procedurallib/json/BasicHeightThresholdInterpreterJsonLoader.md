---
description: Architectural reference for BasicHeightThresholdInterpreterJsonLoader
---

# BasicHeightThresholdInterpreterJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class BasicHeightThresholdInterpreterJsonLoader<K extends SeedResource> extends JsonLoader<K, BasicHeightThresholdInterpreter> {
```

## Architecture & Concepts
The BasicHeightThresholdInterpreterJsonLoader is a specialized, single-purpose deserializer within the procedural generation framework. Its primary function is to translate a specific JSON data structure into a fully realized BasicHeightThresholdInterpreter object.

This class acts as a bridge between static, on-disk configuration and the dynamic, in-memory objects used by the world generation engine. The BasicHeightThresholdInterpreter is a critical component for defining how terrain height influences other generation features, such as biome placement or resource distribution. This loader ensures that the JSON configuration for these height-based rules is parsed, validated, and instantiated correctly at runtime. It is a concrete implementation of the Factory pattern, focused exclusively on one type of procedural component.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level configuration or asset management system when it encounters a JSON object that needs to be interpreted as a height threshold map. The constructor requires the raw JsonElement and other contextual data, indicating it is created with all necessary information to perform its task.
- **Scope:** Extremely short-lived. The object's lifetime is typically confined to a single method call within the broader asset loading pipeline. It is created, its `load` method is invoked, and the resulting BasicHeightThresholdInterpreter is returned.
- **Destruction:** The loader instance becomes eligible for garbage collection immediately after the `load` method completes and its result is captured. It holds no persistent state or external resources that require explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state, consisting of the source JsonElement and associated metadata, is effectively immutable. All state is provided via the constructor and is not modified during the object's lifetime. The class is stateless in an operational sense; its purpose is to transform input to output.
- **Thread Safety:** This class is not thread-safe and is not designed for concurrent use. It is intended to be used by a single thread as part of a sequential asset loading process. There are no internal locks or synchronization primitives. Invoking `load` from multiple threads on a single instance is an unsupported and invalid use case.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | BasicHeightThresholdInterpreter | O(N) | Parses the internal JSON data and constructs a new BasicHeightThresholdInterpreter. Throws IllegalStateException if the required "Positions" or "Values" keys are absent from the JSON. N is the total number of elements in the position and value arrays. |

## Integration Patterns

### Standard Usage
This loader is an internal component of the configuration system. A managing service will identify the relevant JSON section, create the loader, and immediately use it to produce the runtime object.

```java
// A hypothetical configuration manager processing a larger file
JsonElement heightThresholdJson = rootConfig.get("terrainHeightMap");
SeedString seed = ...;
Path dataFolder = ...;
int length = ...;

// The loader is created, used, and then discarded
BasicHeightThresholdInterpreterJsonLoader loader = new BasicHeightThresholdInterpreterJsonLoader(seed, dataFolder, heightThresholdJson, length);
BasicHeightThresholdInterpreter interpreter = loader.load();

// The resulting interpreter is now used by the generation engine
proceduralEngine.registerHeightInterpreter(interpreter);
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not retain and reuse instances of this loader. Each instance is tied to the specific JsonElement provided at construction. For new data, create a new loader.
- **Ignoring Exceptions:** The `load` method throws an unchecked IllegalStateException on malformed or incomplete JSON. This indicates a critical configuration error. Failure to catch this at a high level can lead to world generation failures or engine instability. The exception should be caught, logged with context, and should typically halt the loading process.

## Data Pipeline
This class is a key transformation step in the procedural configuration pipeline. It converts declarative data from a text file into an executable component used by the core engine.

> Flow:
> JSON Configuration File -> Gson Parser -> JsonElement -> **BasicHeightThresholdInterpreterJsonLoader** -> BasicHeightThresholdInterpreter (In-Memory Object) -> Procedural World Generation Engine

