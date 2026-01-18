---
description: Architectural reference for ClimateMaskJsonLoader
---

# ClimateMaskJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.climate
**Type:** Transient Factory

## Definition
```java
// Signature
public class ClimateMaskJsonLoader<K extends SeedResource> extends JsonLoader<K, ClimateMaskProvider> {
```

## Architecture & Concepts
The ClimateMaskJsonLoader is a high-level factory class within the server's world generation pipeline. It serves as an orchestrator, responsible for parsing a climate mask JSON file and constructing a fully configured ClimateMaskProvider object.

This class does not perform low-level parsing itself. Instead, it delegates the responsibility for parsing specific sections of the JSON file—such as noise, climate graphs, and unique zones—to specialized, single-purpose loader classes. This design follows the **Composition over Inheritance** principle, creating a clean separation of concerns where this loader acts as an aggregation point.

Its primary role is to transform a declarative, on-disk data format (JSON) into a complex, in-memory object graph that the world generator can use to make procedural decisions about climate placement. The loader is the critical bridge between static world configuration and the dynamic procedural generation engine.

## Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level world generation service when it needs to load a specific climate definition. The constructor immediately triggers file I/O to read and parse the target JSON file, throwing a fatal Error if the file is missing or malformed. This fail-fast behavior ensures system integrity.
- **Scope:** Short-lived and transactional. The object is designed to be used for a single `load` operation. Its lifespan is typically confined to the method scope in which it was created.
- **Destruction:** Once the `load` method returns the ClimateMaskProvider, the loader instance has fulfilled its purpose and is eligible for garbage collection. It holds no persistent state and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state, consisting of the seed, data folder path, and the parsed JSON element from the file, is established at construction and is effectively immutable. The `load` method is a pure function of this initial state. It does not cache its result; each invocation re-processes the internal JSON data and constructs a new object graph.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for single-threaded, synchronous execution. Instantiating, loading, and discarding the object within the same thread of execution is the only supported operational model.

## API Surface
The public contract is minimal, exposing only the primary factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ClimateMaskProvider | O(N) | Constructs and returns a new ClimateMaskProvider. N is the size of the JSON configuration. Throws fatal errors if required JSON keys are missing. |

## Integration Patterns

### Standard Usage
The loader is instantiated with context (seed, paths), used immediately to produce a provider object, and then discarded.

```java
// Context provided by the world generation orchestrator
SeedString<WorldSeed> worldSeed = ...;
Path dataFolderPath = ...;
Path climateMaskFile = dataFolderPath.resolve("zones/temperate_forest.json");

// Instantiate, load, and capture the result
ClimateMaskJsonLoader<WorldSeed> loader = new ClimateMaskJsonLoader<>(worldSeed, dataFolderPath, climateMaskFile);
ClimateMaskProvider provider = loader.load();

// The provider is now a fully configured, usable object
// The loader instance is no longer needed
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain an instance of ClimateMaskJsonLoader to call `load` multiple times. This is highly inefficient as it rebuilds the entire complex object graph from the parsed JSON on every call.
- **Catching Errors:** The constructor and `load` method can throw a java.lang.Error on file I/O failure or critical parsing issues. This is an intentional design choice to signal a catastrophic and unrecoverable configuration problem. Code should **not** attempt to catch this error; it should be allowed to propagate and halt the server startup to alert developers to the misconfiguration.
- **Concurrent Access:** Do not pass an instance of this loader to another thread. Its internal state is not protected, and concurrent access will lead to undefined behavior.

## Data Pipeline
The class orchestrates a data transformation pipeline, converting a static file into a live engine component.

> Flow:
> JSON File on Disk -> `Files.newBufferedReader` -> `JsonParser` -> `JsonElement` -> **ClimateMaskJsonLoader** -> Delegation to `(ClimateNoiseJsonLoader, ClimateGraphJsonLoader, ...)` -> **ClimateMaskProvider** (In-Memory Object)

