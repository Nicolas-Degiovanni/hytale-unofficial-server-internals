---
description: Architectural reference for CaveGenerator
---

# CaveGenerator

**Package:** com.hypixel.hytale.server.worldgen.cave
**Type:** Transient Service

## Definition
```java
// Signature
public class CaveGenerator {
```

## Architecture & Concepts

The CaveGenerator is the central orchestrator for the procedural generation of all underground cave networks. It operates as a high-level factory that translates abstract configuration, defined in **CaveType** objects, into a concrete, traversable graph of **CaveNode** objects. This resulting graph, encapsulated within a **Cave** object, represents the complete structure of a cave system before it is voxelized into the game world.

The core generation algorithm is a stateful, recursive-descent process. Starting from a single entry point, the generator iteratively adds new cave segments (nodes) by branching off existing ones. Each step of this process is governed by a complex set of rules, probabilities, and environmental constraints defined within the initial CaveType.

Key architectural principles include:

*   **Deterministic Generation:** The entire process is seeded, ensuring that for a given world seed and starting coordinate, the exact same cave system will be generated every time. This is achieved by hashing the world seed with the cave's origin coordinates to create a unique and predictable Random instance for each cave system.
*   **Configuration-Driven:** The behavior, shape, and complexity of a cave are not hardcoded. Instead, they are driven entirely by the data within the provided CaveType, which specifies parameters like entry nodes, child node types, branching factors, and depth limits.
*   **Environmental Awareness:** The generator does not operate in a vacuum. It is tightly integrated with the **ChunkGenerator** to constantly query for biome information. This allows it to enforce rules that prevent caves from generating or continuing into inappropriate biomes, ensuring environmental cohesion.

## Lifecycle & Ownership

*   **Creation:** A CaveGenerator instance is expected to be instantiated once by a higher-level world generation service during server initialization. It is constructed with a complete array of all **CaveType** definitions that the server supports.
*   **Scope:** The instance is session-scoped. It persists for the entire lifetime of the server's world generation pipeline and is used for all cave generation requests within that world.
*   **Destruction:** The class has no explicit destruction or cleanup logic. It is eligible for garbage collection when the owning world generation service is decommissioned, typically during a server shutdown.

## Internal State & Concurrency

*   **State:** The only persistent internal state is the `caveTypes` array, which is provided at construction and treated as immutable. All other state, such as the Random instance and the in-progress Cave object, is confined to the execution scope of a single `generate` method call. The CaveGenerator itself is effectively stateless between top-level calls.
*   **Thread Safety:** The class is re-entrant and designed for concurrent use. The primary public method, `generate`, creates a new `FastRandom` instance for each invocation, ensuring that parallel generation tasks for different cave systems do not interfere with one another.

    **WARNING:** While the CaveGenerator is thread-safe, its dependencies may not be. It makes frequent calls to the provided `ChunkGenerator` instance. The caller must ensure that the `ChunkGenerator` itself is safe for concurrent access from multiple world generation threads. The call to `ChunkGenerator.getResource().getRandom()` within `generatePrefabs` is a potential point of contention if not managed correctly by the resource provider.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generate(seed, chunkGenerator, caveType, x, y, z) | Cave | O(N) | The primary entry point. Generates a complete cave system based on the given parameters. N is the number of nodes in the resulting cave. |
| getCaveTypes() | CaveType[] | O(1) | Returns the array of cave configurations this generator was initialized with. |

## Integration Patterns

### Standard Usage

The CaveGenerator should be managed by a central world generation service. During chunk population, this service determines if a cave system should originate in a given area. If so, it retrieves the appropriate CaveType and invokes the generator.

```java
// WorldGeneratorService.java
// Assume caveGenerator is an initialized instance field

CaveType selectedType = this.findCaveTypeForBiome(currentBiome);
if (selectedType != null) {
    // Generate the logical structure of the cave
    Cave caveSystem = caveGenerator.generate(
        worldSeed,
        this.chunkGenerator,
        selectedType,
        originX,
        originY,
        originZ
    );

    // Pass the generated Cave object to the voxelization pipeline
    voxelizer.rasterize(caveSystem);
}
```

### Anti-Patterns (Do NOT do this)

*   **On-Demand Instantiation:** Do not create a new CaveGenerator for every generation request. This is inefficient as it requires re-passing the CaveType configurations. A single, long-lived instance should be used.
*   **State Leakage:** Do not retain references to the `Random` object or the `Cave` object outside the scope of a single `generate` call. Each generation is a distinct, isolated operation.
*   **Bypassing Biome Checks:** The generator's reliance on the `ChunkGenerator` for biome validation is critical for world consistency. Providing a mock or null `ChunkGenerator` will lead to severe world generation artifacts, such as aquatic caves generating in a desert.

## Data Pipeline

The flow of data through the CaveGenerator is a transformation from a high-level request into a detailed structural graph.

> Flow:
> World Seed + Coordinates -> Hashing Function -> **CaveGenerator.generate()** -> Recursive Node Expansion -> Biome/Height Validation via ChunkGenerator -> Prefab Population -> **Cave Object** -> Voxelization Pipeline

