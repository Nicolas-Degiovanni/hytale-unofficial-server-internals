---
description: Architectural reference for ChunkGeneratorJsonLoader
---

# ChunkGeneratorJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader
**Type:** Transient

## Definition
```java
// Signature
public class ChunkGeneratorJsonLoader extends Loader<SeedStringResource, ChunkGenerator> {
```

## Architecture & Concepts

The ChunkGeneratorJsonLoader is the primary orchestrator for server-side world generation initialization. It serves as the high-level factory responsible for translating a world's on-disk JSON configuration into a fully configured, in-memory ChunkGenerator instance.

This class is not a simple data parser; it is a stateful, multi-stage builder that governs the entire worldgen loading pipeline. It reads and interprets the root `World.json` file, then delegates to a series of specialized, subordinate loaders (e.g., ZonePatternProviderJsonLoader, MaskProviderJsonLoader, ZonesJsonLoader) to process different aspects of the world definition.

Its core architectural role is to resolve all dependencies required by the ChunkGenerator. This includes loading world dimensions, selecting a terrain mask, parsing zone definitions, and establishing the correct data folder paths. It effectively acts as the composition root for the world generation system, assembling disparate components into a cohesive, functional service.

## Lifecycle & Ownership

-   **Creation:** An instance is created by the server's world loading service at the beginning of a world's initialization sequence. It is provided with the world's seed and the root path to its data files.
-   **Scope:** Short-lived and task-specific. The object's lifetime is bound to the execution of its `load` method. It is designed to be used once to produce a single ChunkGenerator.
-   **Destruction:** Once the `load` method returns the ChunkGenerator, the ChunkGeneratorJsonLoader instance has fulfilled its purpose and is eligible for garbage collection. It holds no persistent state that is relevant after initialization.

## Internal State & Concurrency

-   **State:** The loader is stateful during the execution of the `load` method, creating numerous intermediate objects. However, its initial state (seed, dataFolder) is immutable after construction. A critical side effect of its operation is the mutation of the shared SeedStringResource object, which it populates with the resolved PrefabStore and data folder path.
-   **Thread Safety:** **This class is not thread-safe.** The `load` method performs extensive, sequential file I/O and complex object graph construction. It is fundamentally designed to be executed synchronously on a single thread during the server's world startup phase.
    **WARNING:** Concurrent invocation of the `load` method will result in race conditions, I/O exceptions, and a corrupted world state. Do not share instances of this loader across threads.

## API Surface

The public contract is minimal, exposing only the constructor and the primary factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ChunkGeneratorJsonLoader(seed, dataFolder) | constructor | O(1) | Creates a new loader instance for a specific world definition. |
| load() | ChunkGenerator | O(N) | Orchestrates the entire loading process. Throws Error or IllegalArgumentException on file corruption, I/O failure, or invalid configuration. N represents the combined complexity of all world configuration files. |

## Integration Patterns

### Standard Usage

This class is intended for use by the core server engine during world startup. A developer would typically not interact with it directly.

```java
// Invoked by the server's world management system
SeedStringResource seedResource = createSeedResource(worldSeed);
Path worldDataPath = Paths.get("server/worlds/myworld");

// The loader is created, used once, and then discarded
ChunkGeneratorJsonLoader loader = new ChunkGeneratorJsonLoader(seedResource, worldDataPath);
ChunkGenerator generator = loader.load();

// The resulting generator is now used by the world thread
server.getWorldThread().setChunkGenerator(generator);
```

### Anti-Patterns (Do NOT do this)

-   **Instance Reuse:** Do not call the `load` method multiple times on the same instance. This is not the intended lifecycle and will result in redundant, expensive file I/O operations. Create a new loader for each world initialization task.
-   **State Mutation After Construction:** Do not attempt to modify the loader's internal state after it has been constructed. The `dataFolder` and `seed` are considered final.
-   **Manual Construction:** Manually constructing a ChunkGeneratorJsonLoader outside of the server's core world initialization sequence is strongly discouraged. The surrounding engine provides the necessary context and state management that this loader depends on.

## Data Pipeline

The loader transforms on-disk configuration files into a live ChunkGenerator object. The data flow is a sequence of dependent loading and parsing stages.

> Flow:
> 1.  **Input:** `World.json` file path.
> 2.  **Parse World.json:** The loader reads global settings like `Width`, `Height`, `Masks`, and `PrefabStore`.
> 3.  **Delegate Mask Loading:** Based on the `Masks` entry, it instantiates and invokes either a `ClimateMaskJsonLoader` or `MaskProviderJsonLoader` to produce a MaskProvider object.
> 4.  **Delegate Zone Loading:** The loader reads `Zones.json` and creates a `ZonePatternProviderJsonLoader`. This subordinate loader is then used to orchestrate the loading of all individual `Zone` definitions via a `ZonesJsonLoader`.
> 5.  **Assemble Dependencies:** All loaded components (MaskProvider, Zones, PrefabStore, etc.) are collected.
> 6.  **Instantiate:** A new `ChunkGenerator` is created, injecting all resolved dependencies into its constructor.
> 7.  **Output:** A fully configured `ChunkGenerator` instance.

