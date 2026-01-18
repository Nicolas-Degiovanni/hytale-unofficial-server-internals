---
description: Architectural reference for DirectGrid
---

# DirectGrid

**Package:** com.hypixel.hytale.server.worldgen.climate
**Type:** Singleton

## Definition
```java
// Signature
public class DirectGrid implements CellDistanceFunction {
```

## Architecture & Concepts
The DirectGrid class is a foundational strategy component within the server-side procedural world generation engine. It provides a specific, "identity" implementation of the CellDistanceFunction interface.

Its primary architectural role is to **disable cellular logic** for noise functions or procedural algorithms that require it. In procedural generation, CellDistanceFunctions are typically used to create Voronoi patterns, cellular noise, or other grid-based features by finding the nearest point on a lattice. DirectGrid subverts this entirely.

By implementing methods like nearest2D to simply return the input coordinates, it effectively treats every point in space as the center of its own unique cell. This makes it a "pass-through" or "no-op" strategy, ensuring that any subsequent noise sampling or procedural evaluation occurs on an unmodified, continuous coordinate system. It is used in scenarios where a pure, non-cellular noise field is desired for biome placement, climate mapping, or terrain height.

## Lifecycle & Ownership
- **Creation:** Statically instantiated at class load time via the public static final INSTANCE field. This is an eager, thread-safe initialization.
- **Scope:** Application-scoped. The single INSTANCE persists for the entire lifetime of the server process.
- **Destruction:** The object is managed by the JVM's class loader and is cleaned up during process shutdown. No manual destruction is ever required.

## Internal State & Concurrency
- **State:** **Stateless and immutable.** The DirectGrid class contains no instance fields and its behavior is purely functional, depending only on the arguments passed to its methods.
- **Thread Safety:** **Inherently thread-safe.** As a stateless singleton, the single INSTANCE can be safely shared and invoked by any number of world generation threads concurrently without locks or any other synchronization primitives.

## API Surface
The public contract is defined by the CellDistanceFunction interface. The key methods have a trivial implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| nearest2D(...) | void | O(1) | Identity transformation. Copies the input coordinates x and y directly to the output buffer, bypassing any cellular distance calculation. |
| nearest3D(...) | void | O(1) | Identity transformation. Copies the input coordinates x, y, and z directly to the output buffer. |
| transition2D(...) | void | O(1) | No-operation. This method is empty and has no effect. |
| transition3D(...) | void | O(1) | No-operation. This method is empty and has no effect. |
| evalPoint(...) | void | O(1) | No-operation. This method is empty and has no effect. |
| collect(...) | void | O(1) | No-operation. This method is empty and has no effect. |

## Integration Patterns

### Standard Usage
DirectGrid is intended to be used as a strategy object, passed into procedural algorithms that accept a CellDistanceFunction. It is the standard choice when cellular effects are explicitly not desired.

```java
// In a hypothetical biome mapper, DirectGrid is used to sample
// a continuous temperature map without cellular distortion.
BiomeMapper mapper = new BiomeMapper(
    seed,
    noiseGenerator,
    DirectGrid.INSTANCE // Pass the singleton instance
);

Biome biome = mapper.getBiomeAt(x, z);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DirectGrid()`. The class is a singleton designed to have exactly one instance to eliminate object allocation overhead. Always use the provided `DirectGrid.INSTANCE`.
- **Subclassing:** Do not extend DirectGrid. Its behavior is atomic and complete. If alternative distance logic is required, create a new, independent implementation of the CellDistanceFunction interface.

## Data Pipeline
The data flow through DirectGrid is a direct pass-through, making it a computationally inexpensive component in the world generation pipeline.

> Flow:
> Input Coordinates (x, y, z) -> **DirectGrid.nearest3D** -> Output Buffer (containing identical x, y, z) -> Downstream Noise Sampler

