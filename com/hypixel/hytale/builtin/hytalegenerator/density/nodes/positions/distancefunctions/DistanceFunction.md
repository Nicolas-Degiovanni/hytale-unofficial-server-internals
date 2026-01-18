---
description: Architectural reference for DistanceFunction
---

# DistanceFunction

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.distancefunctions
**Type:** Base Class / Strategy

## Definition
```java
// Signature
public abstract class DistanceFunction {
```

## Architecture & Concepts
The DistanceFunction class is an abstract base class that defines a fundamental contract for calculating distance within the world generation system. It serves as the core component of the Strategy Pattern for distance-based calculations, allowing various algorithms (e.g., Euclidean, Manhattan, Chebyshev) to be implemented and used interchangeably by higher-level density nodes.

This abstraction is critical for procedural generation, where complex shapes, biomes, and terrain features are often defined by their relationship to points, lines, planes, or other geometric primitives. By encapsulating the distance calculation logic, the system allows for the flexible composition of generation rules. For example, a sphere can be defined as all points where a DistanceFunction returns a value less than a given radius.

It operates as a pure computational unit within a larger generation graph, taking a 3D coordinate and returning a scalar value representing distance.

### Lifecycle & Ownership
- **Creation:** Concrete implementations of DistanceFunction are instantiated by the procedural generation graph system, typically during the parsing of world generation configuration files or through direct construction by a parent density node. They are not managed as engine-wide services.
- **Scope:** The lifetime of a DistanceFunction instance is tightly coupled to the lifetime of the world generation node that owns it. It is effectively a value object or a configuration parameter for that node.
- **Destruction:** Instances are subject to standard Java garbage collection. They are cleaned up when the parent node that references them is no longer in scope. No explicit destruction or resource cleanup is required.

## Internal State & Concurrency
- **State:** The abstract base class is stateless. Concrete implementations are expected to be immutable. Any parameters required for calculation, such as a center point or a plane normal, should be provided during construction and stored in final fields.
- **Thread Safety:** This class and its intended implementations must be unconditionally thread-safe. The world generation process is heavily multithreaded, and a single DistanceFunction instance may be invoked concurrently from multiple worker threads processing different world chunks. The contract of getDistance implies it is a pure function with no side effects.

**WARNING:** Introducing mutable state into a DistanceFunction subclass is a severe architectural violation and will lead to non-deterministic generation artifacts and race conditions.

## API Surface
The public contract is minimal, consisting of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDistance(Vector3d) | double | O(1) | Calculates the distance from a conceptual origin to the provided 3D point. The definition of "origin" and the algorithm used are determined by the concrete subclass. |

## Integration Patterns

### Standard Usage
A developer typically does not interact with this class directly. Instead, they implement a concrete subclass defining a specific distance metric. This subclass is then used by other components in the generation pipeline.

```java
// Example of a concrete implementation
public class EuclideanDistance extends DistanceFunction {
    private final Vector3d origin;

    public EuclideanDistance(Vector3d origin) {
        this.origin = origin;
    }

    @Override
    public double getDistance(@Nonnull Vector3d point) {
        return this.origin.distance(point);
    }
}

// Usage within a hypothetical density node
DistanceFunction df = new EuclideanDistance(new Vector3d(0, 64, 0));
double density = 1.0 - df.getDistance(currentWorldPosition);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to use `new DistanceFunction()`. The class is abstract and cannot be instantiated.
- **Stateful Implementations:** Do not create subclasses that modify their internal state within the getDistance method. This breaks thread safety and violates the functional contract.
- **Input Mutation:** The input Vector3d to getDistance must not be modified. It should be treated as a read-only parameter.

## Data Pipeline
This class is a computational node, not a data transport layer. It acts as a function within a larger data-flow graph for world generation.

> Flow:
> World Generator -> Density Node -> **DistanceFunction.getDistance(position)** -> double (distance value) -> Further Density Calculation

