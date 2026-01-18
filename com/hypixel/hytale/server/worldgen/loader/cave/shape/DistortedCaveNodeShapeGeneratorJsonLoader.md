---
description: Architectural reference for DistortedCaveNodeShapeGeneratorJsonLoader
---

# DistortedCaveNodeShapeGeneratorJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.cave.shape
**Type:** Transient

## Definition
```java
// Signature
public class DistortedCaveNodeShapeGeneratorJsonLoader extends CaveNodeShapeGeneratorJsonLoader {
```

## Architecture & Concepts
The DistortedCaveNodeShapeGeneratorJsonLoader is a specialized deserializer within the server-side procedural world generation framework. Its primary function is to translate a declarative JSON configuration file into a live, executable `DistortedCaveNodeShapeGenerator` object.

This class operates as a concrete implementation within a factory system. A higher-level service reads a world generation asset and, based on the data schema, dispatches the parsing task to the appropriate loader. This loader is specifically responsible for JSON objects that define cave segments with complex, noise-driven distortions.

It follows a compositional design, delegating the parsing of nested JSON structures (such as numeric ranges or noise parameters) to other, more granular loaders like `DoubleRangeJsonLoader` and `ShapeDistortionJsonLoader`. This approach promotes separation of concerns and allows for reusable parsing logic across the world generation system.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a parent factory or service during the world generation asset loading phase. It is created specifically to handle a single `JsonElement` that conforms to the distorted cave node shape schema.

- **Scope:** This object is **short-lived** and its scope is typically confined to the method call that requires the deserialization. It exists only to execute its `load` method and produce a generator object.

- **Destruction:** The loader instance holds no persistent state or system resources. It becomes eligible for garbage collection immediately after the `load` method returns its result. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The internal state consists of the `seed`, `dataFolder`, and `json` element provided at construction. This state is treated as **immutable** for the lifetime of the object. The loader's methods are purely functional, transforming this input state into a new output object without side effects on the loader itself.

- **Thread Safety:** This class is **conditionally thread-safe**. While a single instance is not designed to be accessed by multiple threads, its transient nature and immutable state make it safe to use in a multi-threaded environment. Separate threads can safely create and use their own instances of this loader without risk of interference, provided the underlying `JsonElement` is not mutated externally.

## API Surface
The public contract is minimal, centered entirely on the `load` factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | CaveNodeShapeEnum.CaveNodeShapeGenerator | O(N) | Deserializes the internal JSON object into a fully configured generator. N is the number of keys in the JSON. Throws `JsonParseException` on malformed data. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay developers. It is invoked by the world generation asset pipeline. A hypothetical parent factory would use it as follows.

```java
// Pseudo-code for a higher-level loading service
JsonElement config = readCaveNodeConfig("my_distorted_cave.json");
SeedString seed = world.getSeed().append("CaveNodes");
Path dataFolder = world.getDataFolderPath();

// The factory creates the correct loader for the config
DistortedCaveNodeShapeGeneratorJsonLoader loader = new DistortedCaveNodeShapeGeneratorJsonLoader(seed, dataFolder, config);

// The loader produces the final, usable generator object
CaveNodeShapeEnum.CaveNodeShapeGenerator generator = loader.load();

// The generator is now used by the cave generation algorithm
caveSystem.addNode(generator);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain an instance of this loader for later use. It is designed for a single, one-shot `load` operation. Create a new instance for each distinct JSON object to be parsed.

- **External State Mutation:** Do not modify the `JsonElement` after passing it to the loader's constructor. The loader assumes its input is stable, and external mutation will lead to undefined behavior.

- **Direct Instantiation:** Manually constructing this class bypasses the world generation system's factory logic. This can lead to inconsistencies if the system later expects a different loader type for a given configuration. Always allow the parent system to select and instantiate the correct loader.

## Data Pipeline
This loader acts as a critical translation step in the data flow from static assets to live world generation logic.

> Flow:
> World Gen Asset (JSON file) -> Gson Parser -> **DistortedCaveNodeShapeGeneratorJsonLoader** -> DistortedCaveNodeShapeGenerator (Live Object) -> Cave Generation Algorithm

