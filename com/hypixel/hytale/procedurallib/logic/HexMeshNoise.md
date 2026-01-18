---
description: Architectural reference for HexMeshNoise
---

# HexMeshNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Transient

## Definition
```java
// Signature
public class HexMeshNoise implements NoiseFunction {
```

## Architecture & Concepts
The HexMeshNoise class is a specialized implementation of the NoiseFunction interface, designed to generate values that form a web-like or mesh-like pattern on a hexagonal grid. It is a fundamental building block within the procedural generation library for creating complex, structured, yet organic-looking features like cave networks, crystalline structures, or biome transition patterns.

Unlike traditional noise algorithms (e.g., Perlin, Simplex) that produce smooth, continuous gradients, HexMeshNoise determines the noise value based on a point's proximity to the nearest line segment in a procedurally generated hexagonal mesh.

The core architectural components it orchestrates are:
- **Hexagonal Grid Logic:** It does not operate on a standard Cartesian grid. All coordinate transformations and distance calculations are delegated to the HexCellDistanceFunction utility, which correctly handles the geometry of a hexagonal tiling.
- **Conditional Density:** The placement of nodes (vertices) in the mesh is not random but deterministic based on a seed. The IIntCondition interface, passed as the *density* parameter, acts as a predicate to decide whether a cell at a given location should contain a node. This allows for powerful control over the mesh's sparseness.
- **Positional Jitter:** To avoid perfectly regular, artificial-looking patterns, the CellJitter component is used to displace each node slightly from its parent cell's center. This introduces a critical element of natural variation.
- **Connectivity Rules:** The boolean flags *linesX*, *linesY*, and *linesZ* control the axes along which connections between nodes are formed, enabling different mesh topologies (e.g., horizontal bands, vertical structures, or a full triangular mesh).

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its public constructor. It is expected to be instantiated and configured by a higher-level system, such as a BiomeGenerator or a WorldFeaturePlacer, which defines the specific parameters for the desired mesh pattern.
- **Scope:** The object's lifetime is tied to its creator. It is a stateful but immutable configuration object that persists as long as the generator that owns it is in scope. It is not a global singleton.
- **Destruction:** The object holds no native resources or managed state requiring explicit cleanup. It is eligible for garbage collection once all references to it are dropped.

## Internal State & Concurrency
- **State:** The internal state of HexMeshNoise is **immutable**. All of its fields, including the density condition, thickness, and jitter strategy, are marked as final and are set exclusively during construction. The object serves as a container for a specific, unchanging noise configuration.
- **Thread Safety:** This class is **unconditionally thread-safe**. Because its internal state is immutable, the primary `get` method is a pure function of its inputs (seed, coordinates). The same instance can be safely shared and called concurrently by multiple worker threads during parallelized world generation without any need for external locking or synchronization.

## API Surface
The public contract is minimal, focusing on configuration at creation and execution via the NoiseFunction interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| HexMeshNoise(density, thickness, jitter, ...) | constructor | O(1) | Configures and initializes a new mesh noise function. |
| get(seed, offsetSeed, x, y) | double | O(1) | Calculates the 2D noise value. The complexity is constant as it only ever checks a fixed set of neighboring cells. |
| get(seed, offsetSeed, x, y, z) | double | N/A | **Unsupported.** Throws UnsupportedOperationException. Do not call. |

## Integration Patterns

### Standard Usage
HexMeshNoise is designed to be composed with other noise functions. A typical use case involves creating a configured instance and passing it to a system that evaluates noise fields.

```java
// A generator configures a specific mesh pattern
IIntCondition densityCheck = new IntThresholdCondition(0.5);
CellJitter jitter = new NoJitter();
NoiseFunction webPattern = new HexMeshNoise(densityCheck, 2.5, jitter, true, true, false);

// Later, the engine evaluates the noise at a specific point
double value = webPattern.get(worldSeed, featureSeed, 1024.5, 768.2);
if (value < -0.8) {
    // Place a "web" block here
}
```

### Anti-Patterns (Do NOT do this)
- **Per-call Instantiation:** Do not create a new HexMeshNoise instance for every call to `get`. The object is lightweight but should be instantiated once per configuration and reused for all subsequent evaluations.
- **Ignoring Coordinate Space:** The *thickness* parameter is scaled internally to the hexagonal grid's domain. Passing pre-scaled values may lead to unexpected visual results.
- **Using the 3D `get` Method:** The 3D variant of the `get` method is not implemented and will crash the calling thread. This implementation is strictly for 2D noise generation.

## Data Pipeline
The flow of data for a single evaluation of the `get` method is a multi-stage process to determine the distance from a point to the generated mesh.

> Flow:
> Input Coordinates (x, y) -> HexCellDistanceFunction scales coordinates -> Grid Cell (cx, cy) is identified -> Neighboring cells are checked -> For each neighbor, a hash is generated from its coordinates and the seed -> The **IIntCondition** (density) predicate is evaluated on the hash -> If true, a node's jittered position is calculated via **CellJitter** -> The distance from the input point to the line segment between adjacent nodes is computed -> The minimum distance across all checked segments is found -> The final distance is normalized to a noise value between -1.0 and 1.0.

