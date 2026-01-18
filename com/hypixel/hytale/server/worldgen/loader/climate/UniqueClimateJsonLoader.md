---
description: Architectural reference for UniqueClimateJsonLoader
---

# UniqueClimateJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.climate
**Type:** Transient

## Definition
```java
// Signature
public class UniqueClimateJsonLoader<K extends SeedResource> extends JsonLoader<K, UniqueClimateGenerator.Entry> {
```

## Architecture & Concepts
The UniqueClimateJsonLoader is a specialized data marshalling component within the server's world generation framework. Its sole responsibility is to translate a specific JSON object definition into a fully realized, in-memory `UniqueClimateGenerator.Entry` object. This class does not contain any world generation logic itself; it is strictly a parser and data hydrator.

Architecturally, this loader acts as a concrete implementation of a Strategy Pattern for data loading. It is one of many such loaders, each designed to handle a specific piece of the world generation configuration. It inherits from the generic `JsonLoader`, which provides foundational utilities for JSON traversal and error handling, ensuring consistency across the loading pipeline.

The existence of this class implies a hierarchical configuration system where a master loader parses a large worldgen file, identifies a "UniqueClimate" definition, and delegates the processing of that specific JSON sub-tree to an instance of UniqueClimateJsonLoader. This promotes separation of concerns, making the overall worldgen configuration system more modular and maintainable.

### Lifecycle & Ownership
- **Creation:** An instance is created dynamically by a parent loading process. It is never instantiated directly by application logic. The constructor's requirement for a pre-parsed `JsonElement` confirms that a higher-level system is responsible for file I/O and initial JSON parsing.
- **Scope:** The object's lifetime is exceptionally short. It exists only for the duration of the `load` method call. It is a single-use, "fire-and-forget" utility.
- **Destruction:** The object is eligible for garbage collection immediately after the `load` method returns its result. It holds no persistent references and is not registered with any service locator.

## Internal State & Concurrency
- **State:** The loader's state (seed, data folder, and the target JSON element) is provided at construction and is treated as immutable for the object's lifespan. It is a stateful transformer, but its state is fixed and confined to a single operation.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed with the expectation of being created, used, and discarded within the confines of a single, synchronous loading thread. Attempting to access a shared instance from multiple threads will lead to unpredictable behavior, as the parent `JsonLoader` may maintain internal, non-atomic state during parsing.

## API Surface
The public contract is minimal, exposing only the primary transformation method. All other methods are protected implementation details.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | UniqueClimateGenerator.Entry | O(1) | Parses the configured JSON element and returns a new `UniqueClimateGenerator.Entry`. Throws an exception if required fields are missing or malformed. |

## Integration Patterns

### Standard Usage
This class is used internally by the world generation system. A developer would not typically interact with it directly. The pattern below illustrates its intended use by a hypothetical parent loader.

```java
// Hypothetical parent loader processing a larger config
JsonElement worldGenConfig = ... // Load and parse entire file
JsonElement uniqueClimateJson = worldGenConfig.getAsJsonObject().get("UniqueClimateDefinition");

// Delegate parsing to the specialized loader
UniqueClimateJsonLoader<MySeed> loader = new UniqueClimateJsonLoader<>(seed, dataPath, uniqueClimateJson);
UniqueClimateGenerator.Entry climateEntry = loader.load();

// Register the resulting object with the world generator
worldGenerator.registerUniqueClimate(climateEntry);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new UniqueClimateJsonLoader()` in general application code. This class is an internal detail of the configuration loading pipeline.
- **Instance Re-use:** Do not attempt to call `load` multiple times on the same instance or modify its internal state after creation. Each JSON object requires a new loader instance.
- **Asynchronous Loading:** Do not pass an instance of this loader to another thread. The entire loading operation it belongs to should be completed synchronously.

## Data Pipeline
This loader is a critical step in transforming declarative configuration on disk into active logic within the world generator.

> Flow:
> WorldGen JSON File -> Master JSON Parser -> `JsonElement` Sub-Tree -> **UniqueClimateJsonLoader** -> `UniqueClimateGenerator.Entry` -> UniqueClimateGenerator Service -> World Generation Algorithm

