---
description: Architectural reference for CoordinateRandomizerJsonLoader
---

# CoordinateRandomizerJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class CoordinateRandomizerJsonLoader<K extends SeedResource> extends JsonLoader<K, ICoordinateRandomizer> {
```

## Architecture & Concepts
The CoordinateRandomizerJsonLoader is a specialized deserializer responsible for transforming a JSON configuration block into a fully realized ICoordinateRandomizer object. It serves as a critical bridge between the declarative, on-disk world generation definitions and the imperative, in-memory procedural generation engine.

This class operates within a hierarchical loading framework. It is not a standalone service but rather a component invoked by a parent parser when it encounters a JSON object that defines a coordinate randomization strategy. Its primary function is to interpret the structure of this JSON, which typically involves a list of noise generators and an optional rotation component.

It embodies a composition pattern by delegating the loading of its constituent parts—individual noise functions and coordinate rotations—to other, more specialized loaders like NoisePropertyJsonLoader and CoordinateRotatorJsonLoader. This creates a modular and maintainable system for parsing complex, nested procedural generation rules.

## Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level configuration loader when it needs to parse a specific JsonElement representing a coordinate randomizer. It is provided with all necessary context, including the seed, data folder path, and the target JSON, upon construction.

- **Scope:** The object's lifetime is extremely short and ephemeral. It exists only for the duration of the call to its load method. It is a single-use, fire-and-forget object.

- **Destruction:** The instance is eligible for garbage collection immediately after the load method returns its result. The created ICoordinateRandomizer object is what persists, not the loader itself.

## Internal State & Concurrency
- **State:** The loader's state consists of the seed, dataFolder, and json element provided at construction. This state is treated as immutable for the object's lifetime and is used exclusively to parameterize the single loading operation. The class does not cache results or modify its internal state after creation.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to operate within a single-threaded asset loading pipeline. No internal locking mechanisms are present, and concurrent access would lead to unpredictable behavior.

## API Surface
The public contract is minimal, exposing only the primary loading operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ICoordinateRandomizer | O(N) | Deserializes the configured JSON into an ICoordinateRandomizer instance. N is the number of generators in the JSON array. Returns EMPTY_RANDOMIZER if the source JSON is null. |

## Integration Patterns

### Standard Usage
This class is intended to be used by other components within the procedural library's loading system. A parent loader instantiates it with a specific JSON fragment and immediately consumes the result.

```java
// A parent system has a JsonObject 'config' and a specific 'randomizerJson' element
SeedString<MySeed> seed = ...;
Path dataFolder = ...;
JsonElement randomizerJson = config.get("myRandomizer");

// Instantiate, load, and use the result. The loader is then discarded.
CoordinateRandomizerJsonLoader<MySeed> loader = new CoordinateRandomizerJsonLoader<>(seed, dataFolder, randomizerJson);
ICoordinateRandomizer randomizer = loader.load();

// The 'randomizer' object is now used by the engine
proceduralEngine.setCoordinateRandomizer(randomizer);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain a reference to the loader after calling load. It is a single-shot utility, and its internal state is not designed for multiple invocations.

- **State Mutation:** Do not attempt to modify the loader's internal state via reflection or other means. The provided constructor arguments are assumed to be constant for the duration of the load operation.

## Data Pipeline
The class acts as a transformation stage in a larger data flow, converting structured text data into a functional object graph.

> Flow:
> JSON File on Disk -> Parent JSON Parser -> JsonElement -> **CoordinateRandomizerJsonLoader** -> (Delegates to NoisePropertyJsonLoader, CoordinateRotatorJsonLoader) -> Assembled ICoordinateRandomizer Object -> Procedural Generation Engine

