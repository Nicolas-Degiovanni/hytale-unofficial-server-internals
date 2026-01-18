---
description: Architectural reference for FluidMaterial
---

# FluidMaterial

**Package:** com.hypixel.hytale.builtin.hytalegenerator.material
**Type:** Value Object

## Definition
```java
// Signature
public class FluidMaterial {
```

## Architecture & Concepts
The FluidMaterial class is an immutable data structure that represents a single fluid voxel within the world. It is a fundamental component of the world generation and fluid simulation systems, acting as a lightweight data carrier.

Its primary role is to encapsulate the identity of a fluid (via fluidId) and its current state (via fluidLevel). Crucially, it also holds a reference back to its originating MaterialCache. This design is central to a **Flyweight pattern**, where the MaterialCache acts as a factory and repository for FluidMaterial instances. By ensuring that only one unique instance of a FluidMaterial exists for any given combination of fluid ID and level, the engine drastically reduces memory overhead, which is critical in a large-scale voxel world.

This class represents the *state* of a fluid at a point in space, not the *behavior*. Logic for fluid dynamics, rendering, and interaction is handled by higher-level systems that consume FluidMaterial objects.

### Lifecycle & Ownership
-   **Creation:** FluidMaterial instances are created exclusively by the MaterialCache factory class. The constructor is intentionally package-private to enforce this pattern. Direct instantiation is a design violation.
-   **Scope:** Instances are ephemeral and context-dependent. They are typically retrieved from the MaterialCache during a specific operation (e.g., a chunk generation pass, a fluid simulation tick) and are expected to be short-lived.
-   **Destruction:** Managed entirely by the Java Garbage Collector. Once an instance is no longer referenced by any part of the world data or an ongoing process, it becomes eligible for collection.

## Internal State & Concurrency
-   **State:** Strictly **immutable**. All member fields (materialCache, fluidId, fluidLevel) are declared final and are set only once at construction time. This guarantees that a FluidMaterial object represents a consistent and unchangeable snapshot of a fluid's state.

-   **Thread Safety:** Inherently **thread-safe**. Due to its immutability, a FluidMaterial instance can be safely shared and read across multiple threads without any need for locks, synchronization, or other concurrency primitives. This is a vital feature for the performance of the parallelized world generator.

## API Surface
The public contract is minimal, focusing on data access and value-based comparisons.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelCache() | MaterialCache | O(1) | Retrieves the parent factory cache that created this instance. |
| equals(Object o) | boolean | O(1) | Performs a value-based equality check. Two instances are equal if their cache, fluid ID, and fluid level are identical. |
| hashCode() | int | O(1) | Computes a hash code based on the fluid ID and level for efficient use in hash-based collections. |
| contentHash(int, byte) | static int | O(1) | Provides a static, consistent hashing function for fluid properties, used internally by hashCode. |

## Integration Patterns

### Standard Usage
A developer should never create a FluidMaterial directly. Instead, it must be retrieved from the appropriate MaterialCache, which is typically accessed via a world or generator context.

```java
// Correctly retrieve a FluidMaterial instance via its factory
MaterialCache cache = world.getMaterialCache();
FluidMaterial flowingWater = cache.getFluidMaterial(WATER_ID, (byte) 4);

// Use the material to set a block in the world
world.setBlock(x, y, z, flowingWater);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** **CRITICAL:** Do not invoke the constructor, even if you are in the same package. Using `new FluidMaterial(...)` bypasses the caching mechanism entirely, leading to excessive memory allocation and breaking reference equality checks that other systems may rely on.
-   **State Modification:** Do not attempt to modify a FluidMaterial instance via reflection. The engine's stability relies on the immutability of these objects. To represent a change in fluid level, a *new* FluidMaterial instance must be retrieved from the MaterialCache.

## Data Pipeline
FluidMaterial is a data product of the MaterialCache factory and a data source for various engine systems.

> Flow:
> World Generation Algorithm → MaterialCache.getFluidMaterial(id, level) → **FluidMaterial** (Instance) → Voxel Data Structure (e.g., Chunk) → Fluid Simulation & Mesh Generation Systems

