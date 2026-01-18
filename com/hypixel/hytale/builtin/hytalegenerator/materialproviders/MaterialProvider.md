---
description: Architectural reference for MaterialProvider
---

# MaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class MaterialProvider<V> {
```

## Architecture & Concepts
The MaterialProvider is a foundational abstraction within the procedural world generation system. It serves as the core component for a **Strategy Pattern**, defining a contract for any object that can decide which material, represented by the generic type V, should be placed at a specific voxel coordinate.

Its primary role is to decouple the high-level world generation algorithm (which iterates through voxels and calculates environmental properties) from the low-level, biome-specific material placement logic. The generator prepares a comprehensive `Context` object for each voxel, containing rich environmental data such as 3D position, terrain density, and proximity to biome edges. This context is then passed to a concrete MaterialProvider implementation, which uses this information to return a specific material type.

This design allows for a highly modular and composable generation pipeline. Different biomes or geological layers can simply plug in different MaterialProvider implementations without altering the core generation loop. The generic parameter V enhances this flexibility, allowing the same abstraction to be used for providing block types, biome identifiers, or any other per-voxel data.

### Lifecycle & Ownership
- **Creation:** Concrete implementations of MaterialProvider are instantiated and configured during the world generation bootstrap phase. They are typically defined within biome or world configuration files and loaded by a central generation service. They are not intended for dynamic creation during a generation pass.
- **Scope:** The lifetime of a MaterialProvider instance is tied to the scope of the configuration it belongs to, such as a specific biome definition. It is not a global singleton and persists only as long as its parent configuration is loaded.
- **Destruction:** Instances are eligible for garbage collection once the world generation process that required them is complete and their parent configuration is unloaded.

## Internal State & Concurrency
- **State:** The abstract MaterialProvider class is stateless. However, concrete implementations are expected to be stateful, holding configuration data such as references to other providers for sub-materials (e.g., a stone provider, a dirt provider).
- **Thread Safety:** **CRITICAL:** Implementations of MaterialProvider **must be immutable** after their initial configuration. The world generation engine is heavily multi-threaded, and a single provider instance will be invoked concurrently by multiple worker threads to generate different parts of a world chunk. All per-voxel, mutable data is passed via the `Context` object to ensure thread safety without the need for expensive locking. Any internal state must be considered read-only during the `getVoxelTypeAt` call.

## API Surface
The public contract is minimal, focusing entirely on the material resolution logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelTypeAt(Context) | V | O(N) | Abstract method to determine the material for a given context. Returns null if no material should be placed. Complexity is implementation-dependent. |
| noMaterialProvider() | MaterialProvider | O(1) | Static factory for a Null Object Pattern implementation. Returns a provider that always resolves to null. |

## Integration Patterns

### Standard Usage
A MaterialProvider is never used in isolation. It is invoked by a higher-level generation service which is responsible for constructing the `Context` object for each voxel in a chunk.

```java
// Within a world generator loop for a given position...
Vector3i currentPosition = new Vector3i(x, y, z);

// 1. The generator calculates all environmental data.
double density = terrainDensityProvider.getDensity(x, y, z);
int depth = calculateDepth(x, y, z);
// ... and so on for all context fields.

// 2. A comprehensive context is created.
MaterialProvider.Context context = new MaterialProvider.Context(
    currentPosition,
    density,
    depth,
    // ... other parameters
);

// 3. The configured provider is invoked to get the final material.
V material = biome.getMaterialProvider().getVoxelTypeAt(context);

// 4. The material is placed in the world.
chunk.setVoxel(x, y, z, material);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Never modify the internal state of a MaterialProvider from within the `getVoxelTypeAt` method. This will violate thread safety and lead to non-deterministic, unpredictable world generation.
- **Ignoring Context:** Do not implement a provider that ignores the incoming `Context` object to query global state or other services. This breaks the architectural pattern and makes generation results dependent on invocation order. The `Context` is the single source of truth for the voxel being processed.
- **Expensive Computations:** Avoid performing expensive calculations like noise lookups or density calculations inside a provider. This work should be done once by the parent generator and passed in via the `Context`. The provider's role is to make a decision based on pre-computed data.

## Data Pipeline
The MaterialProvider is a key decision point in the voxel data pipeline for world generation.

> Flow:
> Voxel Coordinate Request -> World Generator (Calculates Density, Depth, etc.) -> **MaterialProvider.Context** (Data is packaged) -> **MaterialProvider.getVoxelTypeAt()** (Decision is made) -> Material Type (V) -> Chunk Data Structure

---
description: Architectural reference for MaterialProvider.Context
---

# MaterialProvider.Context

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public static class Context {
```

## Architecture & Concepts
The MaterialProvider.Context is a parameter object that encapsulates all known environmental information about a single voxel being processed by the world generator. Its existence is critical to the performance and thread safety of the MaterialProvider system.

By bundling all necessary data into this single object, the system achieves several key goals:
1.  **Decoupling:** MaterialProvider implementations do not need to know about the broader generation system. They are given all the data they need to make a decision.
2.  **Efficiency:** Expensive environmental calculations (like terrain density, distance fields, or vertical space analysis) are performed only once by the main generation loop. The results are then passed to potentially many different providers in a chain or tree, avoiding redundant computation.
3.  **Clarity:** The method signature for `getVoxelTypeAt` remains clean and stable, even as new environmental parameters are added to the generation process. New data is simply added as a field to the Context object.

## Lifecycle & Ownership
- **Creation:** A new Context object is instantiated by the world generation loop for **every single voxel** that is processed. This is a high-frequency allocation.
- **Scope:** The object's lifetime is extremely short. It is created, passed to one or more MaterialProvider instances, and is then immediately eligible for garbage collection once the `getVoxelTypeAt` call returns.
- **Destruction:** Handled automatically by the Java Garbage Collector. Given its high allocation rate, the performance of this object is critical.

## Internal State & Concurrency
- **State:** The Context is a mutable data container. Its fields are public and directly accessible for high performance.
- **Thread Safety:** The object is **not thread-safe**. This is by design. Each worker thread in the generation engine creates and operates on its own private instance of the Context for the specific voxel it is processing. It is never shared across threads.

## API Surface
The class has no methods, only public fields and constructors. It serves purely as a data structure.

| Symbol | Type | Description |
| :--- | :--- | :--- |
| position | Vector3i | The absolute world coordinate of the voxel. |
| density | double | The density value at this position, typically from a noise function. Used to determine if a voxel is solid or air. |
| depthIntoFloor | int | Vertical distance from the first solid block below. 0 if this is the first solid block. |
| depthIntoCeiling | int | Vertical distance from the first solid block above. 0 if this is the first solid block. |
| spaceAboveFloor | int | Vertical distance to the first solid block above. |
| spaceBelowCeiling | int | Vertical distance to the first solid block below. |
| workerId | WorkerIndexer.Id | Identifier for the worker thread processing this voxel. Can be used for deterministic, thread-local random number generation. |
| terrainDensityProvider | TerrainDensityProvider | A reference to the density provider, allowing for more complex density queries if absolutely necessary. |
| distanceToBiomeEdge | double | The distance to the nearest edge of the current biome. Useful for creating transition effects. |

## Integration Patterns

### Standard Usage
The Context is created and populated by a generator and immediately passed to a provider.

```java
// Correct usage: create, populate, use, and discard.
MaterialProvider.Context ctx = new MaterialProvider.Context(
    pos, density, depth, ceiling, ...
);
V material = provider.getVoxelTypeAt(ctx);
```

### Anti-Patterns (Do NOT do this)
- **Caching or Reusing:** Do not cache or reuse a Context object for a different voxel. Its data is specific to a single coordinate and reusing it will lead to incorrect world generation.
- **Sharing Across Threads:** Never pass a Context instance from one worker thread to another. This is a severe violation of the concurrency model.

