---
description: Architectural reference for Assignments
---

# Assignments

**Package:** com.hypixel.hytale.builtin.hytalegenerator.propdistributions
**Type:** Strategy Contract / Factory

## Definition
```java
// Signature
public abstract class Assignments {
```

## Architecture & Concepts
The Assignments class is an abstract contract that forms the core of the procedural prop distribution system within Hytale's world generator. It embodies the **Strategy** design pattern, defining an algorithm for selecting a specific Prop to place at a given world coordinate.

This class acts as a decision-making engine. For any point in 3D space, an implementation of Assignments determines which entity (e.g., a tree, rock, or bush) should be assigned to it. It is a critical link between the high-level biome definition and the low-level block and entity placement logic.

The inputs to its core method, propAt, include not only the position but also a WorkerIndexer.Id and the distance to the biome edge. This design indicates a sophisticated, multi-threaded generation architecture where placement decisions can be influenced by worker-specific seeds (to ensure deterministic randomness) and spatial context (proximity to biome borders).

The static factory method, noPropDistribution, provides a null-object implementation, ensuring that systems requiring an Assignments instance can function correctly even in zones with no props, preventing null pointer exceptions and simplifying generator logic.

## Lifecycle & Ownership
- **Creation:** Concrete implementations are instantiated by higher-level world generation controllers, typically when a specific biome or world zone is being processed. They are often configured and created based on asset definitions for that biome. The static noPropDistribution factory is used for zones that are explicitly defined to be empty of props.
- **Scope:** An Assignments instance is transient and scoped to a specific generation task. It is not a global singleton. Its lifetime is tied to the processing of a particular world region or biome.
- **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection once the world generation task that created it completes and releases its reference. There are no manual destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The abstract class itself is stateless. However, concrete implementations are expected to be stateful, holding configuration data such as lists of potential Props, noise function parameters, and placement rules. This state is intended to be **immutable** after the object's construction to guarantee deterministic and thread-safe execution.
- **Thread Safety:** **Implementations of this class must be thread-safe.** The propAt method is designed to be called concurrently from multiple world generation worker threads, as evidenced by the WorkerIndexer.Id parameter. All internal state must be read-only during generation. Any mutable state within the propAt method would violate the system's design and lead to severe generation artifacts and race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| propAt(Vector3d, WorkerIndexer.Id, double) | Prop | O(1) | The core decision-making function. Returns the specific Prop assigned to a world position. Computationally intensive but constant time. |
| getRuntime() | int | O(1) | Retrieves an identifier for the generator runtime configuration. Used for versioning or debugging. |
| getAllPossibleProps() | List<Prop> | O(1) | Returns a complete list of all Props that this distribution *could* potentially place. Primarily used for asset pre-loading and validation. |
| noPropDistribution(int) | Assignments | O(1) | A static factory that returns a shared, inert implementation that always assigns Prop.noProp(). |

## Integration Patterns

### Standard Usage
An Assignments instance is retrieved from a biome or zone configuration. The world generator then iterates over coordinates, invoking propAt for each position to determine which prop, if any, should be placed.

```java
// A world generator obtains an Assignments strategy for the current biome
Assignments propStrategy = currentBiome.getPropAssignments();
WorkerIndexer.Id workerId = currentThread.getWorkerId();

// For a given coordinate in the world
Vector3d targetPosition = new Vector3d(128, 64, 256);
double distanceToEdge = calculateDistanceToBiomeEdge(targetPosition);

// The strategy is invoked to make a placement decision
Prop assignedProp = propStrategy.propAt(targetPosition, workerId, distanceToEdge);

// The generator acts on the decision
if (!assignedProp.isNoProp()) {
    world.placeProp(targetPosition, assignedProp);
}
```

### Anti-Patterns (Do NOT do this)
- **Mutable State:** Implementing a subclass where propAt modifies internal fields is a critical error. This will break determinism and cause severe concurrency issues, resulting in a corrupted world.
- **Ignoring Worker ID:** Failing to use the WorkerIndexer.Id to seed noise functions or random number generators will create visible seams and artifacts at the boundaries of chunks processed by different threads.
- **Expensive Computation:** While some complexity is expected, the propAt method should not perform file I/O, network requests, or excessively long calculations, as it is on the critical path for world generation performance.

## Data Pipeline
The flow of data is unidirectional, transforming a spatial query into a concrete entity assignment.

> Flow:
> World Generator Query (at Vector3d) -> **Assignments.propAt()** -> Internal Noise Functions & Rule Evaluation -> Selected Prop -> World Generator -> Prop Instantiation in World Data

