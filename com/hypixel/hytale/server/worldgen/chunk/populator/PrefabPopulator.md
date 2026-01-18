---
description: Architectural reference for PrefabPopulator
---

# PrefabPopulator

**Package:** com.hypixel.hytale.server.worldgen.chunk.populator
**Type:** Transient Service

## Definition
```java
// Signature
public class PrefabPopulator {
```

## Architecture & Concepts

The **PrefabPopulator** is a critical stage in the server-side world generation pipeline. It is responsible for placing pre-designed structures, known as prefabs, into the world after the primary terrain and biome data have been generated for a chunk. This class acts as the orchestrator that translates the declarative placement rules defined in biome and zone configurations into concrete block placements.

The core design follows a deterministic, multi-phase process for each chunk generation task:

1.  **Collection:** The system first identifies all potential prefabs that could influence the current chunk. This involves scanning not only the biomes within the chunk but also those in an extended surrounding region, as large prefabs may originate in a neighboring chunk but overlap into the current one.
2.  **Candidate Evaluation:** For each potential prefab, a series of conditions are evaluated based on the world seed and coordinates. These include noise density maps, height constraints, and parent block requirements to determine if a prefab *can* spawn at a given location. Validated locations produce a **Candidate** record.
3.  **Conflict Resolution:** All generated candidates are evaluated for spatial overlap. Using a priority system defined by the **PrefabCategory**, the system de-conflicts intersecting prefabs. When two prefabs overlap, the one with the lower priority value (higher importance) is kept, and the other is discarded. This ensures that important structures are not overwritten by more common, lower-priority decorations.
4.  **Generation:** Finally, the surviving, non-conflicted candidates are passed to the **PrefabPasteUtil** to be rendered into the chunk's block buffer. This includes both biome-specific procedural prefabs and world-specific "unique" prefabs which have fixed locations.

This entire process is stateless from the perspective of the caller; all necessary information is derived from the world seed and the provided **ChunkGeneratorExecution** context.

### Lifecycle & Ownership
- **Creation:** A single instance of **PrefabPopulator** is instantiated at server startup and managed by a central resource provider, accessible via `ChunkGenerator.getResource()`. It is not intended to be created directly.
- **Scope:** The object instance persists for the entire server session. However, its internal state—such as the lists of biomes, candidates, and conflicts—is strictly transient. This state is populated at the beginning of a `run` call and completely cleared at the end, making the object reusable for subsequent chunk population tasks.
- **Destruction:** The object is destroyed and garbage collected during server shutdown.

## Internal State & Concurrency
- **State:** The **PrefabPopulator** is highly stateful *during* the execution of a `run` operation. It maintains mutable lists of biomes and prefab candidates, along with a **BitSet** for tracking placement conflicts. This state is ephemeral and is reset upon completion of the operation.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be executed by a single world generation thread at a time. The internal fields are mutated extensively throughout the `run` method. Concurrent execution would result in race conditions and severe world generation corruption. All calls must be synchronized externally.

## API Surface

The primary entry point is the static `populate` method, which retrieves the shared instance and invokes its `run` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| populate(seed, execution) | static void | O(N\*M) | Orchestrates the entire prefab population process for a given chunk. N is the number of potential prefabs in the search area; M is the number of candidates. |

## Integration Patterns

### Standard Usage

The **PrefabPopulator** is invoked by the core world generation system, typically within the **ChunkGenerator**, after initial terrain has been shaped. It is not intended for general use by gameplay logic.

```java
// System-level call within a ChunkGenerator task
// The execution object contains all necessary context for the chunk.

int worldSeed = ...;
ChunkGeneratorExecution execution = ...;

PrefabPopulator.populate(worldSeed, execution);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PrefabPopulator()`. The system relies on a shared, reused instance to reduce object allocation overhead. Always use the static `populate` method.
- **Concurrent Execution:** Never call `populate` from multiple threads simultaneously. The internal state is not protected by locks and will become corrupted. World generation tasks that use this class must be queued and executed serially.
- **State Manipulation:** Do not attempt to access or modify the internal fields of the shared **PrefabPopulator** instance. Its state is managed exclusively by the `run` method.

## Data Pipeline

The flow of data through the populator is a funnel, starting with broad environmental context and narrowing down to a final set of prefabs to be placed.

> Flow:
> Chunk Generation Task -> **PrefabPopulator.populate()**
> 1.  **Collect Biomes**: Scans `ChunkGeneratorExecution` and surrounding zone data to build a list of relevant **Biome** objects.
> 2.  **Collect Prefabs**: Iterates biomes, reads `PrefabContainer` rules, and uses `PrefabPatternGenerator` to create a list of potential `Candidate` structs.
> 3.  **Collect Conflicts**: Analyzes the `Candidate` list for bounding box intersections and culls lower-priority candidates.
> 4.  **Generate Prefabs**: Feeds the final, de-conflicted `Candidate` list into `PrefabPasteUtil`.
> 5.  `PrefabPasteUtil` writes final block data into the `ChunkGeneratorExecution` buffer.

