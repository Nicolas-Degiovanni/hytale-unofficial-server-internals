---
description: Architectural reference for FloatRangeJsonLoader
---

# FloatRangeJsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Transient

## Definition
```java
// Signature
public class FloatRangeJsonLoader<K extends SeedResource> extends JsonLoader<K, IFloatRange> {
```

## Architecture & Concepts
The FloatRangeJsonLoader is a specialized, short-lived parser responsible for deserializing a JSON representation into a concrete IFloatRange object. It serves as a crucial component within the procedural generation asset pipeline, providing a robust and flexible bridge between human-readable configuration files and the engine's internal data structures.

Its primary architectural significance lies in its tolerance for multiple JSON formats for defining a numeric range. A game designer or developer can specify a range as a single number, a two-element array, or a structured object with Min and Max keys. This flexibility simplifies configuration management without sacrificing precision.

Furthermore, the loader incorporates a transformation hook via the FloatToFloatFunction functional interface. This allows the calling system to inject custom logic—such as scaling, clamping, or unit conversion—that is applied transparently during the parsing process. This decouples the raw data values in the JSON files from their final, in-memory representation used by the procedural engine.

## Lifecycle & Ownership
- **Creation:** An instance is created on-demand by a higher-level configuration loader or asset parser whenever it encounters a JSON field that must be resolved into an IFloatRange. It is not managed by a central registry or dependency injection framework.

- **Scope:** The object's lifetime is exceptionally brief. It is designed to be instantiated, used for a single call to its load method, and then immediately discarded. Its scope is typically confined to the local method where the parsing occurs.

- **Destruction:** The object becomes eligible for garbage collection as soon as the load method returns. It holds no native resources or persistent state that would require explicit cleanup.

## Internal State & Concurrency
- **State:** The FloatRangeJsonLoader is **effectively immutable**. All of its internal fields, including default values, the source JsonElement, and the transformation function, are finalized during construction. The load method is a pure function that operates solely on this immutable state.

- **Thread Safety:** This class is **inherently thread-safe**. Due to its immutable design, a single instance can be safely passed between threads and its load method can be invoked concurrently without any risk of data corruption or race conditions. No external locking is required.

## API Surface
The primary public contract is the load method, which executes the parsing logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | IFloatRange | O(1) | Deserializes the configured JsonElement into an IFloatRange instance. Throws IllegalStateException if the source JSON is malformed or violates structural rules (e.g., an array with 3 elements). |

## Integration Patterns

### Standard Usage
The loader should be instantiated with the source JsonElement and any required defaults. The load method is then called to produce the final IFloatRange object.

```java
// Assume 'configObject' is a JsonObject read from a file
// and 'seed' is the current procedural context.
JsonElement rangeElement = configObject.get("treeHeight");

// Instantiate the loader with a default range of 10.0 to 20.0
FloatRangeJsonLoader loader = new FloatRangeJsonLoader(
    seed,
    dataFolderPath,
    rangeElement,
    10.0f,
    20.0f
);

// The loaded range is now ready for use by the procedural engine.
IFloatRange treeHeight = loader.load();
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not cache or reuse FloatRangeJsonLoader instances. They are lightweight, stateful parsers designed to be single-use. Reusing an instance to parse a different JsonElement is not possible and retaining instances unnecessarily increases memory pressure.

- **Ignoring Exceptions:** The load method signals data format errors by throwing IllegalStateException. This is a critical failure that indicates a problem with the source asset files. This exception must be caught and handled by the asset loading system to prevent the engine from running with corrupt or invalid configuration.

- **Over-reliance on Defaults:** While the loader gracefully handles null or missing JSON by using default values, this can mask underlying issues in configuration files. Systems should treat the use of defaults as a fallback, not the primary operational path.

## Data Pipeline
The FloatRangeJsonLoader operates as a specific step in the broader asset deserialization pipeline. It transforms a generic JSON data structure into a specialized, type-safe engine object.

> Flow:
> JSON File on Disk -> Gson Parser -> JsonElement -> **FloatRangeJsonLoader** -> IFloatRange (In-Memory Object) -> Procedural Generation Algorithm

