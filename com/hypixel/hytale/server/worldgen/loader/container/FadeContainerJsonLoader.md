---
description: Architectural reference for FadeContainerJsonLoader
---

# FadeContainerJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader.container
**Type:** Transient

## Definition
```java
// Signature
public class FadeContainerJsonLoader extends JsonLoader<SeedStringResource, FadeContainer> {
```

## Architecture & Concepts
The FadeContainerJsonLoader is a specialized deserializer within the server-side world generation framework. Its sole responsibility is to translate a specific JSON configuration structure into a strongly-typed FadeContainer object. This class acts as a schema-aware parser, bridging the gap between declarative world generation data files and the in-memory objects used by procedural algorithms.

It operates as a component in a larger, hierarchical loading system, evidenced by its inheritance from the generic JsonLoader. The parent class provides the core JSON traversal and error-handling capabilities, while FadeContainerJsonLoader implements the domain-specific logic for interpreting keys like *FadeStart* and *FadeLength*.

The use of a SeedString in its constructor is a critical design choice. It ensures that the loading process is integrated into the world's deterministic procedural generation pipeline. Any randomization or derived values required during loading can be tied to the world seed, guaranteeing reproducibility.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level world generation orchestrator or asset loading service. It is created specifically to process a single JsonElement that represents a fade container configuration. This is not a long-lived service or singleton.
- **Scope:** The object's lifecycle is exceptionally short and is scoped to a single `load` operation. An instance is created, its `load` method is invoked once, and the resulting FadeContainer is returned. The loader instance is then immediately eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. The class holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The class is stateful but effectively immutable after its construction. It holds references to the initial `seed`, `dataFolder`, and the target `json` element. This state is read-only and is used exclusively to produce the final FadeContainer object.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for single-threaded, one-shot execution. The internal state of the parent JsonLoader is not guaranteed to be safe for concurrent access. While multiple instances can be safely used on different threads to parse different JSON files in parallel, a single instance must be confined to the thread that created it.

## API Surface
The public contract is minimal, exposing only the functionality required to execute the loading operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | FadeContainer | O(1) | Deserializes the internal JSON element into a new FadeContainer instance. Provides default values for any missing optional keys. |

## Integration Patterns

### Standard Usage
The loader is intended to be used by the world generation framework as part of a larger asset loading sequence. The calling system is responsible for acquiring the JSON data and providing it to the loader.

```java
// Assume 'worldGenOrchestrator' has access to the JSON data and seed
SeedString<SeedStringResource> currentSeed = worldGenOrchestrator.getCurrentSeed();
Path dataPath = worldGenOrchestrator.getDataPath();
JsonElement fadeConfigJson = worldGenOrchestrator.getFadeConfiguration();

// The loader is created for a single, specific task
FadeContainerJsonLoader loader = new FadeContainerJsonLoader(currentSeed, dataPath, fadeConfigJson);

// The result is retrieved and the loader is no longer needed
FadeContainer fadeParams = loader.load();

// Use the strongly-typed object in the generation algorithm
terrainGenerator.applyFade(fadeParams);
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not retain a reference to a FadeContainerJsonLoader instance to call `load` multiple times. It is designed for one-shot use. Create a new instance for each distinct JSON element you need to process.
- **State Modification:** Do not attempt to modify the loader's internal state via reflection or other means after construction. Its behavior is only guaranteed with the initially provided state.
- **Cross-Thread Sharing:** Never pass an instance of this loader to another thread. If parallel loading is required, each thread must create its own loader instance with its own JSON data.

## Data Pipeline
This component functions as a specific step in a larger data deserialization and processing pipeline for world generation.

> Flow:
> World Gen Asset File (.json) -> File System Reader -> JSON Parser (Gson) -> JsonElement -> **FadeContainerJsonLoader** -> FadeContainer Object -> World Generation Algorithm

