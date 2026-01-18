---
description: Architectural reference for ListPositionProvider
---

# ListPositionProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.positionproviders
**Type:** Transient Data Provider

## Definition
```java
// Signature
public class ListPositionProvider extends PositionProvider {
```

## Architecture & Concepts
The ListPositionProvider is a concrete implementation of the PositionProvider strategy. Its role within the world generation framework is to supply a finite, predetermined set of coordinates. Unlike procedural providers that calculate positions based on algorithms (e.g., grids, random distributions), this class operates on a static, in-memory list of vectors.

This component is fundamental for declarative world generation, where features, structures, or points of interest must be placed at exact, pre-authored locations. It acts as a data container that adapts a simple list of coordinates to the broader PositionProvider-based generation pipeline.

Internally, it maintains two parallel lists for integer and double-precision vectors. This dual representation ensures that precision is handled correctly during the conversion from one vector type to another, although the primary generation callback, positionsIn, exclusively operates on the double-precision list.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively via the static factory methods *from3i* or *from3d*. The constructor is private to prevent direct instantiation and ensure that every instance is correctly initialized with a valid list of positions. It is typically instantiated by a higher-level configuration or a generator that reads a set of predefined locations.
- **Scope:** The object's lifetime is ephemeral. It is designed to be a short-lived data carrier for a single generation operation. It holds no global state and is not registered as a service.
- **Destruction:** The object is eligible for garbage collection as soon as the calling generator or process that created it completes its operation and discards the reference. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** The internal state consists of two lists of vectors. This state is populated once during creation by copying the contents of the input list. After construction, the state is **effectively immutable**, as no public methods exist to modify the internal lists. This design makes the object a safe and predictable data shuttle.
- **Thread Safety:** The class is **conditionally thread-safe**. Because its internal state is read-only after construction, an instance can be safely passed between and read by multiple threads simultaneously. The creation process itself is not internally synchronized, but the factory methods produce a new, distinct instance each time, eliminating shared mutable state concerns.

**WARNING:** While the instance is safe for concurrent reads, the original list passed to the factory method is not protected. Modifying the source list after creating the provider will have no effect on the provider's internal state, as it operates on a defensive copy.

## API Surface
The public API is minimal, focusing entirely on creation and the fulfillment of the PositionProvider contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| from3i(List<Vector3i>) | ListPositionProvider | O(N) | Static factory. Creates an instance from a list of integer vectors. Copies the list. |
| from3d(List<Vector3d>) | ListPositionProvider | O(N) | Static factory. Creates an instance from a list of double-precision vectors. Copies the list. |
| positionsIn(Context) | void | O(N) | Iterates through all stored positions and passes them to the context consumer. |

## Integration Patterns

### Standard Usage
The primary use case is to feed a known set of locations into a system that expects a PositionProvider.

```java
// 1. Define a list of specific locations
List<Vector3i> structureLocations = new ArrayList<>();
structureLocations.add(new Vector3i(100, 64, 250));
structureLocations.add(new Vector3i(150, 70, 300));

// 2. Create the provider from the list
PositionProvider provider = ListPositionProvider.from3i(structureLocations);

// 3. Pass the provider to a generator or processor
// The generator will call provider.positionsIn(...) internally.
worldGenerator.placeStructures(provider);
```

### Anti-Patterns (Do NOT do this)
- **Large Datasets:** Avoid using this provider for extremely large sets of coordinates (e.g., millions of points). The entire list is held in memory, which can lead to significant heap allocation and performance degradation. For dense or large-scale procedural placement, use an algorithmic provider.
- **Direct Instantiation:** The constructor is private for a reason. Do not use reflection to bypass the static factory methods, as this can result in a partially initialized object.

## Data Pipeline
The data flow is linear: an external collection of vectors is ingested by the provider, which then streams those same vectors to a consumer upon request.

> Flow:
> `List<Vector3d>` -> **ListPositionProvider.from3d()** -> Instance with internal copy -> `positionsIn(context)` -> `context.consumer.accept(vector)` -> Downstream Generation Logic

