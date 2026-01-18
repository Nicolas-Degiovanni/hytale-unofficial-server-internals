---
description: Architectural reference for ClimateTypeJsonLoader
---

# ClimateTypeJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.climate
**Type:** Transient

## Definition
```java
// Signature
public class ClimateTypeJsonLoader<K extends SeedResource> extends JsonLoader<K, ClimateType> {
```

## Architecture & Concepts
The ClimateTypeJsonLoader is a specialized, transient parser responsible for deserializing a JSON definition into a fully realized ClimateType object graph. It serves as a critical component in the server-side world generation pipeline, translating human-readable climate configuration files into the in-memory data structures used by the procedural generation engine.

Architecturally, this class embodies the **Composite** and **Builder** patterns. It processes a single JSON element representing a climate, but also recursively instantiates new loaders for any nested "Children" definitions, effectively building a tree of ClimateType objects.

Furthermore, it acts as a coordinator, delegating the parsing of complex sub-objects to other dedicated loaders, such as ClimateColorJsonLoader and ClimatePointJsonLoader. This separation of concerns ensures that each loader has a single, well-defined responsibility.

## Lifecycle & Ownership
- **Creation:** An instance is created by a higher-level world generation service or, more commonly, by a parent ClimateTypeJsonLoader when it encounters a "Children" array within a JSON file. It is never managed by a dependency injection container or service registry.

- **Scope:** The object's lifetime is exceptionally short and is strictly confined to the execution scope of the `load` method that requires it. It is created, used to produce a single ClimateType object, and then immediately becomes eligible for garbage collection.

- **Destruction:** The object is destroyed by the Java Garbage Collector once the `load` method returns and no more references to the loader exist. It does not implement any special cleanup logic.

## Internal State & Concurrency
- **State:** The internal state of the loader (seed, dataFolder, json, parent) is provided at construction and is treated as immutable for the duration of its lifecycle. The class is fundamentally stateless in that it performs a pure transformation from input JSON to an output object without retaining any information between invocations.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded execution within the world generation process. The absence of locks or other synchronization primitives is intentional, as the transient lifecycle model obviates the need for them.

## API Surface
The public contract is minimal, exposing only the primary `load` method. Protected methods are considered internal implementation details for building the object graph.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | ClimateType | O(N) | Deserializes the configured JSON into a ClimateType object graph. Complexity is linear, proportional to the total number of child climates and points in the JSON structure. Throws runtime exceptions on malformed or missing JSON data. |

## Integration Patterns

### Standard Usage
A developer should never need to interact with this class directly unless extending the world generation system. Its instantiation is an internal detail of the climate loading process, typically invoked recursively.

```java
// Conceptual example of recursive invocation within the system
// Assume 'childJson' is a JsonObject from a "Children" array and 'parentType' is the ClimateType being constructed.
ClimateTypeJsonLoader<MySeed> loader = new ClimateTypeJsonLoader<>(this.seed, this.dataFolder, childJson, parentType);
ClimateType childClimate = loader.load(); // The result is a single node in the climate graph.
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not retain an instance of ClimateTypeJsonLoader and attempt to call `load` on it multiple times. Each instance is configured for a specific JSON element and is not designed for re-use.

- **External Instantiation:** Avoid creating a ClimateTypeJsonLoader from application-level code. The world generation asset pipeline is responsible for initiating the loading process. Manual creation risks providing an incorrect or incomplete context (seed, parent, data folder).

- **Concurrent Access:** Never pass an instance to another thread. The object's state is not protected, and concurrent access will lead to unpredictable behavior and likely data corruption.

## Data Pipeline
This loader functions as a deserialization step, transforming raw data from disk into a usable runtime object for the world generator.

> Flow:
> Climate Asset File (.json) -> Server Asset Pipeline -> **ClimateTypeJsonLoader** -> In-Memory ClimateType Graph -> Biome Placement Engine

