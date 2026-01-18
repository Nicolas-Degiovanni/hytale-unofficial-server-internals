---
description: Architectural reference for ClimatePointJsonLoader
---

# ClimatePointJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.climate
**Type:** Transient

## Definition
```java
// Signature
public class ClimatePointJsonLoader<K extends SeedResource> extends JsonLoader<K, ClimatePoint> {
```

## Architecture & Concepts
The ClimatePointJsonLoader is a specialized deserializer within the server's world generation framework. Its sole responsibility is to translate a JSON data structure into a strongly-typed ClimatePoint object. This class acts as a critical bridge between human-readable world configuration files and the in-memory data models used by procedural generation algorithms.

By extending the generic JsonLoader, it inherits foundational JSON parsing capabilities and error handling logic. This class provides the specific schema and business logic for what constitutes a valid climate point, mapping keys like *Temperature* and *Intensity* to the corresponding fields in the ClimatePoint object. It is a key component in the data-driven design of the world generator, allowing designers to define biome characteristics in external files without modifying engine code.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by higher-level configuration managers or biome loaders when a climate definition needs to be processed from a JSON source. It is never managed as a persistent service.
- **Scope:** The object's lifetime is exceptionally short. It is created for the single purpose of executing the *load* method. Once the resulting ClimatePoint object is returned, the loader instance is immediately eligible for garbage collection.
- **Destruction:** Managed automatically by the Java Garbage Collector. There are no manual cleanup or resource release requirements.

## Internal State & Concurrency
- **State:** The state of the loader is established at construction and is effectively immutable. It holds references to the source JSON data and context paths but does not cache the result of the load operation or modify its own state during execution. Each invocation of the *load* method performs a fresh deserialization.
- **Thread Safety:** This class is **not thread-safe** and is not intended for concurrent access. It is designed to be instantiated and used within the scope of a single thread. Given its transient nature, thread safety is achieved by convention: create, use, and discard the object in a single, non-concurrent operation.

## API Surface
The public contract is minimal, exposing only the primary loading functionality.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ClimatePoint | O(1) | Deserializes the source JSON into a new ClimatePoint object. Throws runtime exceptions if required JSON keys are missing and no defaults are available. |

## Integration Patterns

### Standard Usage
This loader is typically used by a broader system responsible for parsing larger configuration files, such as a biome definition. The system extracts the relevant JSON sub-tree and passes it to the loader for processing.

```java
// A higher-level service obtains the JSON element for climate data
JsonElement climateJson = biomeDefinition.get("climate");
SeedString seed = ...;
Path dataFolder = ...;

// The loader is instantiated for a single, transient operation
ClimatePointJsonLoader loader = new ClimatePointJsonLoader(seed, dataFolder, climateJson);
ClimatePoint climate = loader.load();

// The resulting ClimatePoint object is now used by the world generator
worldGenerator.applyClimate(climate);
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not retain and reuse a ClimatePointJsonLoader instance. It holds no state post-load and is inexpensive to create. Caching provides no performance benefit and complicates lifecycle management.
- **Concurrent Modification:** Do not access a single loader instance from multiple threads. The underlying generic JsonLoader provides no guarantees of thread safety.

## Data Pipeline
The ClimatePointJsonLoader functions as a specific transformation step in the world generation asset pipeline. It converts raw, structured text data into a usable engine data type.

> Flow:
> Biome Definition File (.json) -> GSON Parser -> JsonElement -> **ClimatePointJsonLoader** -> ClimatePoint Object -> Biome Generation Service

