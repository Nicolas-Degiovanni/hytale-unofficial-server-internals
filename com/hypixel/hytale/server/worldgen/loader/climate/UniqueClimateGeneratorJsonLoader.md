---
description: Architectural reference for UniqueClimateGeneratorJsonLoader
---

# UniqueClimateGeneratorJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.climate
**Type:** Transient

## Definition
```java
// Signature
public class UniqueClimateGeneratorJsonLoader<K extends SeedResource> extends JsonLoader<K, UniqueClimateGenerator> {
```

## Architecture & Concepts
The UniqueClimateGeneratorJsonLoader is a specialized parser component within the server's world generation framework. Its primary function is to act as a factory, translating a specific JSON data structure—a JSON array of climate definitions—into a fully realized, in-memory UniqueClimateGenerator object.

This class embodies the **Composite** design pattern. It does not parse the granular details of each climate entry itself. Instead, it orchestrates the loading process by iterating over a collection and delegating the parsing of each individual element to a more specialized loader, the UniqueClimateJsonLoader. This separation of concerns ensures that the logic for parsing a collection is distinct from the logic for parsing a single item, promoting modularity and maintainability.

It serves as a critical bridge between the on-disk configuration of the world (defined in JSON) and the runtime systems that use this configuration to generate terrain and biomes.

### Lifecycle & Ownership
- **Creation:** An instance is created on-demand by a higher-level configuration or asset management system when it encounters a JSON file defining a list of unique climates. The caller is responsible for parsing the source file into a Gson JsonArray and passing it into the constructor.
- **Scope:** The object's lifecycle is extremely short and transactional. It exists only for the duration of the `load` method call. It is a single-use, transient object.
- **Destruction:** The instance holds no persistent state or external resources. It becomes eligible for garbage collection immediately after the `load` method returns the constructed UniqueClimateGenerator.

## Internal State & Concurrency
- **State:** The internal state, consisting of the source JsonArray and contextual paths, is established at construction and is treated as immutable for the object's lifetime. The class is stateful only in the context of a single, atomic load operation.
- **Thread Safety:** This class is **not thread-safe** and is intended for single-threaded access. It performs no internal locking or synchronization. It must not be shared across threads, as its design assumes a synchronous, uninterrupted load operation. The caller is responsible for ensuring that the provided JsonArray is not mutated by another thread during the load process.

## API Surface
The public contract is minimal, exposing only the primary factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | UniqueClimateGenerator | O(N) | Orchestrates the parsing of the internal JSON array. N is the number of climate entries in the array. Returns a shared, static EMPTY instance for an empty array. |

## Integration Patterns

### Standard Usage
This loader is intended to be used as part of a larger data loading sequence. A controlling system parses a JSON file and passes the relevant array to this loader to produce a runtime object.

```java
// Assume 'climateArray' is a JsonArray parsed from a worldgen file
// Assume 'seed' and 'dataFolder' are provided by the context

// 1. Instantiate the loader with the raw JSON data
UniqueClimateGeneratorJsonLoader loader = new UniqueClimateGeneratorJsonLoader<>(seed, dataFolder, climateArray);

// 2. Execute the load operation to get the runtime object
UniqueClimateGenerator climateGenerator = loader.load();

// 3. Use the resulting object in the world generator
worldGenerator.setUniqueClimates(climateGenerator);
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not retain an instance of this loader after calling `load`. It is designed for a single transaction and holds a reference to a specific JsonArray.
- **External Iteration:** Do not attempt to access the internal array and iterate it manually. The `load` method encapsulates the necessary delegation to the correct sub-loader (UniqueClimateJsonLoader) and is the only supported pathway.

## Data Pipeline
This class is a transformation stage in the world generation asset pipeline. It converts structured text data into a functional, in-memory game object.

> Flow:
> Worldgen JSON File -> Gson Parser -> JsonArray -> **UniqueClimateGeneratorJsonLoader** -> UniqueClimateGenerator Object -> World Generation Engine

