---
description: Architectural reference for Material
---

# Material

**Package:** com.hypixel.hytale.builtin.hytalegenerator.material
**Type:** Transient Value Object

## Definition
```java
// Signature
public final class Material {
```

## Architecture & Concepts
The Material class is a fundamental, immutable data structure within the world generation and simulation engine. It serves as a composite value object that represents the complete state of a single voxel in the world grid. Each Material instance encapsulates a pair of constituent parts: a **SolidMaterial** (e.g., stone, dirt, wood) and a **FluidMaterial** (e.g., air, water, lava).

This composite design is critical for correctly modeling game mechanics where solids and fluids can coexist in the same block space. It acts as the primary data carrier for algorithms related to terrain generation, block physics, lighting, and rendering.

A key architectural feature is its built-in performance optimization for hash-based collections. The class employs lazy calculation and caching for its hash codes. This is vital for performance in systems that store vast amounts of block data in HashMaps or HashSets, as it avoids re-computing hashes repeatedly for the same object.

## Lifecycle & Ownership
-   **Creation:** Instances are created on-demand by higher-level systems, typically within world generation pipelines or block update handlers. There is no central factory or registry for Material objects; they are instantiated directly via the constructor.
-   **Scope:** The lifecycle of a Material instance is expected to be short and transient. They are created, used for a calculation or stored in a collection for a specific scope (like a single chunk generation task), and then become eligible for garbage collection.
-   **Destruction:** Managed entirely by the Java garbage collector. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** The core state, defined by the SolidMaterial and FluidMaterial references, is **immutable** due to the use of final fields. However, the class contains mutable internal state in the form of two private `Hash` objects used for caching the results of `hashCode()` and `hashMaterialIds()`. This makes the object technically mutable, although its logical value never changes.

-   **Thread Safety:** This class is **not thread-safe**. The lazy initialization of the internal hash caches is not synchronized.

    **WARNING:** If multiple threads access `hashCode()` or `hashMaterialIds()` on a shared, un-initialized instance of Material concurrently, a race condition will occur. This can lead to the hash being calculated multiple times, with unpredictable results for the `isCalculated` flag. It is designed to be used within a single-threaded context, such as a dedicated world-generation worker thread. Any cross-thread usage requires external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| solid() | SolidMaterial | O(1) | Returns the solid component of the material. |
| fluid() | FluidMaterial | O(1) | Returns the fluid component of the material. |
| hashCode() | int | O(1) | Calculates and returns a hash code based on the object identity of the solid and fluid components. The result is cached after the first call. |
| hashMaterialIds() | int | O(1) | Calculates and returns a hash code based on the numeric IDs of the solid and fluid components. The result is cached after the first call. |

## Integration Patterns

### Standard Usage
A Material should be instantiated to represent the state of a block at a specific coordinate. It is commonly used as a key in maps that store chunk data or as a value to be passed to rendering or physics subsystems.

```java
// In a world generation context
SolidMaterial stone = materialRegistry.getSolid("core:stone");
FluidMaterial air = materialRegistry.getFluid("core:air");
Material stoneBlock = new Material(stone, air);

// Store this material in a chunk data structure
chunk.setMaterialAt(x, y, z, stoneBlock);
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Hashing:** Do not share a Material instance across threads and have them both call `hashCode()` or `hashMaterialIds()` for the first time without external locking. This will cause a race condition on the internal cache.
-   **Excessive Instantiation:** Avoid creating new Material objects inside tight loops for common combinations (e.g., Stone and Air). The calling system should cache and reuse these common instances to reduce memory allocation and garbage collector pressure.

## Data Pipeline
The Material class is not a processing stage in a pipeline but rather the data payload that flows through it. It is typically created early in the world generation process and consumed by subsequent stages.

> Flow:
> Noise Function & Biome Rules -> **Material** (Instance created) -> Chunk Voxel Buffer -> Voxel Mesher -> Render Command Buffer

