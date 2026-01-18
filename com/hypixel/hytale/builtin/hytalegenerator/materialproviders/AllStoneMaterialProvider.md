---
description: Architectural reference for AllStoneMaterialProvider
---

# AllStoneMaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public class AllStoneMaterialProvider extends MaterialProvider<SolidMaterial> {
```

## Architecture & Concepts
The AllStoneMaterialProvider is a concrete implementation of the **Strategy Pattern**, conforming to the MaterialProvider contract. Its role within the world generation pipeline is to supply a foundational, non-procedural material layer for a world chunk.

This provider operates on a simple binary logic: a voxel is either solid stone or empty air. It makes this determination based on a single input from the MaterialProvider.Context: the depth of the voxel relative to the world's surface level. This makes it one of the most primitive and computationally inexpensive providers in the system.

Architecturally, it serves several key purposes:
1.  **Base Layer:** In a composite generator, it can act as the final, fallback layer, ensuring that any unassigned voxels deep underground are filled with a default solid material.
2.  **Debugging & Testing:** It provides a predictable and simple world structure, ideal for testing other game systems like physics, lighting, or AI without the complexity of procedural terrain.
3.  **Performance Baseline:** Its minimal overhead makes it a useful benchmark for measuring the performance impact of more complex, noise-based material providers.

It relies on dependency injection for a reference to the MaterialCache, from which it retrieves singleton instances of ROCK_STONE and EMPTY_AIR materials.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level world generator or biome configuration service during the setup phase of a world generation task. It is never managed as a global singleton.
-   **Scope:** The object's lifetime is strictly bound to the generation task for which it was created. It is a short-lived, transient object.
-   **Destruction:** The instance becomes eligible for garbage collection as soon as the world generator completes its work on a given region and releases its reference to the provider. There are no manual cleanup or disposal requirements.

## Internal State & Concurrency
-   **State:** This class is effectively **immutable** post-construction. Its only internal state is a final reference to the MaterialCache. It performs no internal caching and its behavior is entirely determined by the arguments passed to its methods.
-   **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable nature and lack of side effects, a single instance can be safely shared and invoked by multiple world generation worker threads simultaneously without requiring any locks or synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelTypeAt(Context context) | SolidMaterial | O(1) | Returns a material based on depth. Returns ROCK_STONE if below the surface, otherwise EMPTY_AIR. Throws NullPointerException if the context or its internal state is invalid. |

## Integration Patterns

### Standard Usage
This provider is designed to be composed within a larger world generation system. It is instantiated with a shared MaterialCache and used by a generator to fill in a base layer of terrain.

```java
// Within a world generator or configuration service
MaterialCache sharedCache = context.getService(MaterialCache.class);
MaterialProvider baseProvider = new AllStoneMaterialProvider(sharedCache);

// The generator's core loop then uses the provider
// for each voxel in a given column.
for (int y = 0; y < CHUNK_HEIGHT; y++) {
    MaterialProvider.Context voxelContext = buildContextForVoxel(x, y, z);
    SolidMaterial material = baseProvider.getVoxelTypeAt(voxelContext);
    chunk.setMaterialAt(x, y, z, material);
}
```

### Anti-Patterns (Do NOT do this)
-   **Misuse as a Primary Generator:** Do not use this provider as the sole source of materials for a playable world. It generates a perfectly flat, uninteresting world and is intended only as a foundational layer or a debugging tool.
-   **Redundant Instantiation:** Avoid creating a new AllStoneMaterialProvider for every single voxel. An instance should be created once per generation task and reused for all voxels within that task's scope.
-   **Null Dependency Injection:** The constructor is annotated with Nonnull. Passing a null MaterialCache will not cause an immediate failure but will lead to a NullPointerException during the first call to getVoxelTypeAt. The system expects this dependency to be valid.

## Data Pipeline
The AllStoneMaterialProvider is a component in the broader world generation data flow. It acts as a simple, stateless function that transforms positional data into a material type.

> **Flow:**
> World Generator requests material for a voxel coordinate -> Generator creates a **MaterialProvider.Context** (with depth info) -> Context is passed to **AllStoneMaterialProvider.getVoxelTypeAt()** -> Provider returns a **SolidMaterial** instance -> Generator places the material into a Voxel Chunk data structure

