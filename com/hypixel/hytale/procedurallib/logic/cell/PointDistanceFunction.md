---
description: Architectural reference for PointDistanceFunction
---

# PointDistanceFunction

**Package:** com.hypixel.hytale.procedurallib.logic.cell
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface PointDistanceFunction {
```

## Architecture & Concepts
The PointDistanceFunction interface is a foundational component of the procedural generation library, specifically for algorithms based on cellular noise, point grids, and Voronoi diagrams. It embodies the **Strategy** design pattern, abstracting the mathematical concept of a distance metric.

This abstraction allows the core procedural generation engine to remain agnostic to the specific method used for calculating distance. By providing different implementations of this interface, developers can fundamentally alter the visual and structural characteristics of generated features without modifying the high-level generation algorithms. For example, swapping a Euclidean distance implementation for a Manhattan distance one can change the output from rounded, organic shapes to blocky, artificial-looking structures.

The interface provides two primary methods, one for 2D space and one for 3D space, along with default implementations that expose a richer set of contextual parameters. These extended parameters (seed, cell coordinates, etc.) are hooks for advanced, non-uniform distance functions that might vary based on location or other procedural inputs.

## Lifecycle & Ownership
As an interface, PointDistanceFunction does not have a lifecycle of its own. The lifecycle pertains to its concrete implementations.

- **Creation:** Implementations are typically stateless utility classes (e.g., EuclideanDistance, ManhattanDistance). They are often instantiated as singletons or flyweights at the start of a generation process and injected into the relevant generator services, such as a CellularNoiseGenerator.
- **Scope:** The scope of an implementation object is tied to the generation task it is used for. In most cases, a single, shared instance can persist for the application's lifetime due to its inherent statelessness.
- **Destruction:** Implementations hold no resources and are managed by the Java Garbage Collector. There is no explicit destruction or cleanup required.

## Internal State & Concurrency
- **State:** The interface contract implicitly requires all implementations to be **stateless**. A distance function should produce the same output for the same input coordinates, regardless of when or how many times it is called. Storing state within an implementation is a severe violation of the design contract.
- **Thread Safety:** Implementations **must be** thread-safe. The procedural generation engine is heavily parallelized and will invoke these distance functions from multiple worker threads simultaneously. Because implementations are expected to be stateless, they are inherently thread-safe.

**WARNING:** Creating a stateful or non-thread-safe implementation of PointDistanceFunction will lead to severe and difficult-to-debug generation artifacts, including non-deterministic output and visual tearing between world chunks.

## API Surface
The public contract is focused entirely on calculating distance.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| distance2D(double dx, double dy) | double | O(1) | Calculates the 2D distance for the given deltas. |
| distance3D(double dx, double dy, double dz) | double | O(1) | Calculates the 3D distance for the given deltas. |
| distance2D(int seed, ..., double deltaY) | double | O(1) | Default method providing extended context. Forwards to the simpler `distance2D` by default. |
| distance3D(int seed, ..., double deltaZ) | double | O(1) | Default method providing extended context. Forwards to the simpler `distance3D` by default. |

## Integration Patterns

### Standard Usage
A concrete implementation of PointDistanceFunction is provided to a generator or algorithm that requires a distance metric. The generator then calls the distance methods within its core processing loop.

```java
// A generator receives a distance function via its constructor or a method call
PointDistanceFunction metric = new EuclideanDistance(); // An example implementation
CellularNoiseGenerator generator = new CellularNoiseGenerator(seed, metric);

// The generator internally uses the metric to calculate noise values
double value = generator.getValue(x, y, z);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not store any mutable state within a class that implements PointDistanceFunction. The function must be pure.
- **Ignoring Performance:** The distance methods are called in tight loops, often millions of times for a single generation task. An inefficient implementation (e.g., one with high algorithmic complexity or excessive object allocation) will severely degrade world generation performance.
- **Incorrect Metric Properties:** Ensure your implementation adheres to the mathematical properties of a metric if the consuming algorithm depends on them (e.g., the triangle inequality). Violating these can cause unpredictable algorithmic behavior.

## Data Pipeline
This component is not part of a data pipeline but rather a functional component in a control flow. It acts as a pluggable calculator.

> Flow:
> CellularNoiseGenerator requests value for point (x,y,z) -> It identifies the nearest grid points -> It calculates deltas (dx, dy, dz) -> **PointDistanceFunction.distance3D(dx, dy, dz)** is invoked -> A `double` distance is returned -> The generator uses this distance to determine the final noise value.

