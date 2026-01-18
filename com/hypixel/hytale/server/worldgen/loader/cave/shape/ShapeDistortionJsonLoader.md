---
description: Architectural reference for ShapeDistortionJsonLoader
---

# ShapeDistortionJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.cave.shape
**Type:** Transient

## Definition
```java
// Signature
public class ShapeDistortionJsonLoader<K extends SeedResource> extends JsonLoader<K, ShapeDistortion> {
```

## Architecture & Concepts
The ShapeDistortionJsonLoader is a specialized deserialization component within the server's procedural world generation framework. Its sole responsibility is to parse a specific JSON object and construct a corresponding in-memory ShapeDistortion object. This class operates as a node in a larger, hierarchical loading system, where parent loaders delegate the parsing of specific JSON keys to specialized child loaders like this one.

This loader embodies a key design principle of the world generation system: **composition over inheritance for configuration**. Instead of defining cave shapes in rigid code, the system defines them in data (JSON). This class acts as the bridge, translating a "ShapeDistortion" data block into a functional object that the cave generation algorithms can utilize.

It orchestrates the loading of its constituent parts—Width, Floor, and Ceiling noise profiles—by delegating to another specialized loader, NoisePropertyJsonLoader. This creates a clear separation of concerns, where each loader understands only its own small piece of the configuration schema.

## Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level loader (e.g., a CaveDefinitionLoader) when it encounters a "ShapeDistortion" key during the recursive parsing of a world generation JSON file. It is provided with the specific JsonElement it is responsible for, along with contextual information like the world seed and data folder path.

- **Scope:** The lifecycle of a ShapeDistortionJsonLoader is extremely brief. It is a **transient** or **one-shot** object, designed to exist only for the duration of a single `load` operation.

- **Destruction:** The instance is eligible for garbage collection immediately after the `load` method returns its result to the calling loader. No long-term references to it are maintained.

## Internal State & Concurrency
- **State:** The internal state of the loader consists of immutable references to the `seed`, `dataFolder`, and `json` element provided at construction. The object itself is stateless in the sense that it performs no mutation and caches no data between method calls.

- **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent access. It is intended to be created, used, and discarded within the confines of a single thread, typically the main world generation thread. Attempting to share an instance across multiple threads will lead to unpredictable behavior and is a severe violation of its design contract.

## API Surface
The public contract is minimal, exposing only the functionality required by the parent loading system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ShapeDistortion | O(N) | Parses the configured JSON element and constructs a new ShapeDistortion object. N is the number of defined properties (Width, Floor, Ceiling). Throws exceptions on malformed JSON. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is invoked automatically by the world generation framework's deserialization pipeline. A parent loader would use it as follows.

```java
// Hypothetical usage within a parent JsonLoader
JsonElement distortionJson = rootConfig.get("shapeDistortion");
ShapeDistortionJsonLoader<MySeedType> loader = new ShapeDistortionJsonLoader<>(currentSeed, dataFolderPath, distortionJson);
ShapeDistortion distortion = loader.load();

// The resulting 'distortion' object is then passed to the cave generation algorithm.
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain a reference to this loader for later use. Each instance is tied to a specific JsonElement and is not designed to be re-purposed.
- **Manual Instantiation:** Manually constructing this class outside the established JSON loading pipeline is an anti-pattern. The framework guarantees that the necessary context (seed, data paths) is correctly propagated. Bypassing the framework can lead to initialization errors or incorrect world generation.

## Data Pipeline
This loader is a critical step in transforming static configuration data into a live object used by the procedural generation engine.

> Flow:
> WorldGen JSON File -> Parent Loader -> JsonElement for "ShapeDistortion" -> **ShapeDistortionJsonLoader** -> In-Memory ShapeDistortion Object -> Cave Generation Algorithm

