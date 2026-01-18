---
description: Architectural reference for JsonLoader
---

# JsonLoader

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Framework Component

## Definition
```java
// Signature
public abstract class JsonLoader<K extends SeedResource, T> extends Loader<K, T> {
```

## Architecture & Concepts
The JsonLoader is an abstract base class that forms a foundational component of the procedural asset loading pipeline. It is not intended for direct instantiation but serves as a template for concrete loader implementations that parse JSON data to construct game-ready objects.

Its primary architectural role is to provide a robust, safe, and flexible interface for interacting with JSON data sources. It standardizes the process of reading properties, handling type conversions, and managing errors.

A key architectural feature is the **File Indirection** mechanism. The loader can transparently resolve and load nested JSON files referenced within a parent file. If a JSON object contains a key named "File", the loader will treat its value as a path to another JSON file, load it, and return its contents instead. This powerful pattern allows developers to compose complex configurations from smaller, reusable, and more manageable JSON snippets.

The class promotes a fail-fast design philosophy through its `mustGet` family of methods, which immediately throw an Error if a required property is missing or has an incorrect type, preventing latent data corruption bugs downstream.

### Lifecycle & Ownership
- **Creation:** A JsonLoader is never created directly. A concrete subclass (e.g., a `ModelLoader`) is instantiated by a higher-level system, such as an asset manager or resource registry, when a specific asset needs to be loaded from a JSON definition.
- **Scope:** The lifetime of a JsonLoader instance is transient and scoped to a single loading operation. It is created, used to parse a JSON source and build an object, and then discarded.
- **Destruction:** The object holds no persistent engine resources and is eligible for garbage collection as soon as the loading method returns the final constructed object.

## Internal State & Concurrency
- **State:** The core internal state is the `json` field, which holds a GSON JsonElement representing the parsed data. This field is `final`, so the reference to the JSON tree is immutable after construction. The class performs read-only operations on this tree.
- **Thread Safety:** This class is **not thread-safe**. It performs file I/O and relies on GSON's non-thread-safe object model. It is designed to be used exclusively within a single-threaded resource loading context. Accessing a shared instance from multiple threads will result in undefined behavior and potential data corruption.

## API Surface
The public API is designed for subclasses to use during the loading process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| has(name) | boolean | O(1) | Checks for the existence of a top-level property in the JSON object. |
| get(name) | JsonElement | O(N) | Retrieves a property. If the property is an object with a "File" key, triggers a file load (I/O bound). |
| getRaw(name) | JsonElement | O(1) | Retrieves a property without triggering the "File" indirection mechanism. |
| mustGetObject(key, default) | JsonObject | O(1) | Extracts a required JsonObject. Throws Error on type mismatch or if missing and no default is provided. |
| mustGetArray(key, default) | JsonArray | O(1) | Extracts a required JsonArray. Throws Error on type mismatch or if missing and no default is provided. |
| mustGetString(key, default) | String | O(1) | Extracts a required String. Throws Error on type mismatch or if missing and no default is provided. |
| mustGetBool(key, default) | Boolean | O(1) | Extracts a required Boolean. Throws Error on type mismatch or if missing and no default is provided. |
| mustGetNumber(key, default) | Number | O(1) | Extracts a required Number. Throws Error on type mismatch or if missing and no default is provided. |

## Integration Patterns

### Standard Usage
A concrete implementation extends JsonLoader, calls the super constructor, and uses the provided helper methods to parse the JSON and construct the final object.

```java
// Example of a concrete loader for a hypothetical "Zone" object
public class ZoneLoader extends JsonLoader<ZoneSeed, Zone> {

    public ZoneLoader(ZoneSeed seed, Path dataFolder, JsonElement json) {
        super(seed, dataFolder, json);
    }

    @Override
    public Zone load() {
        // Use helper methods to safely parse the JSON
        String zoneName = mustGetString("name", "Default Zone");
        JsonArray biomes = mustGetArray("biomes", null); // Fails if "biomes" is missing
        
        // ... logic to process biomes array ...

        return new Zone(zoneName, /* ... parsed biome data ... */);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Circular File References:** Creating a "File" indirection that points back to the original file or forms a cycle (e.g., A.json references B.json, and B.json references A.json) will cause a StackOverflowError during loading.
- **Concurrent Access:** Do not share a JsonLoader instance across multiple threads. All loading operations must be confined to a single thread.
- **Ignoring Fail-Fast:** Do not wrap `mustGet` calls in broad try-catch blocks to suppress errors. The purpose of these methods is to halt execution on invalid data to ensure system integrity.

## Data Pipeline
The JsonLoader sits in the middle of the asset loading process, transforming a raw JSON data stream into a structured, type-safe object.

> Flow:
> File System (.json) -> Java NIO `Files` -> GSON `JsonReader` -> GSON `JsonElement` -> **JsonLoader** -> `mustGet` Parsing -> Concrete Game Object (e.g., Zone)

