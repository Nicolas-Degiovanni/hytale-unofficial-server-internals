---
description: Architectural reference for ReturnType
---

# ReturnType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.returntypes
**Type:** Abstract Strategy

## Definition
```java
// Signature
public abstract class ReturnType {
```

## Architecture & Concepts
The ReturnType class is an abstract base class that defines a fundamental strategy within Hytale's procedural world generation system. It establishes the contract for calculating a scalar distance value at a specific point in 3D space, which is a core operation in density function evaluation.

This class is a component of the Strategy design pattern. Concrete implementations (e.g., EuclideanDistance, ChebyshevDistance) provide specific algorithms for the abstract `get` method. These strategies are plugged into higher-level *Position Nodes* within the density graph, allowing world designers to declaratively switch distance calculation methods without altering the graph's structure. Its primary role is to translate positional information into a single double value that other nodes in the density graph can consume to determine material, shape, or other world features.

## Lifecycle & Ownership
- **Creation:** Instances of ReturnType subclasses are not created directly in game logic. They are instantiated by the Density Graph deserializer or builder during the world generation setup phase, based on configuration files (e.g., JSON definitions).
- **Scope:** The lifetime of a ReturnType object is bound to the lifetime of its parent *Position Node* within the in-memory density graph. It persists as long as the world generation configuration is loaded.
- **Destruction:** The object is marked for garbage collection when the density graph is unloaded, typically after a world generation task is complete or the server/client shuts down.

## Internal State & Concurrency
- **State:** The class holds a single piece of mutable state: `maxDistance`. This field acts as a configurable clamping value, preventing calculations from considering points beyond a certain range. It defaults to Double.MAX_VALUE.
- **Thread Safety:** This class is **not thread-safe** for mutation. The `setMaxDistance` method is unsynchronized. The `get` method's thread safety is dependent on the concrete implementation.

    **WARNING:** The design assumes a "write-once, read-many" pattern. The `maxDistance` field should be configured *only* during the single-threaded initialization phase. Modifying this state while the density graph is being evaluated by multiple worker threads will lead to severe race conditions and non-deterministic world generation. All subclass implementations of `get` must be re-entrant and free of side effects to support parallel execution.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(...) | abstract double | Varies | Calculates a distance value based on input vectors and context. This is the primary "hot path" method. |
| setMaxDistance(double) | void | O(1) | Configures the maximum distance threshold. Throws IllegalArgumentException for negative values. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by typical game systems. It is used exclusively by the density graph evaluation engine. The engine configures the object upon creation and then calls the `get` method repeatedly for different world coordinates.

A conceptual example of how the *engine* would use a concrete implementation:

```java
// Engine-level code during world generation setup
// This would be loaded from a config file in reality
EuclideanDistanceReturnType distanceCalc = new EuclideanDistanceReturnType();
distanceCalc.setMaxDistance(256.0);

// The PositionNode would hold a reference to distanceCalc
PositionNode node = new PositionNode(distanceCalc);

// ... later, during multithreaded chunk generation ...
// The engine invokes the node, which delegates to the ReturnType
double densityValue = node.evaluate(x, y, z, context);
```

### Anti-Patterns (Do NOT do this)
- **Post-Initialization Mutation:** Never call `setMaxDistance` after the density graph has been initialized and passed to the generation workers. This will break concurrency guarantees.
- **Stateful Implementations:** Do not create subclasses of ReturnType that store state modified within the `get` method. This will produce incorrect results when executed in parallel.

## Data Pipeline
The ReturnType class is a processing step within the larger density function pipeline. It consumes spatial data and produces a scalar value.

> Flow:
> Density Graph Evaluator -> Provides (Coordinates, Context) -> **ReturnType.get()** -> Returns (Calculated Distance) -> Consumed by other Density Nodes

