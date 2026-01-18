---
description: Architectural reference for CylinderDensity
---

# CylinderDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class CylinderDensity extends Density {
```

## Architecture & Concepts
CylinderDensity is a foundational node within the procedural world generation framework. It does not represent a physical, solid cylinder but rather a mathematical density field whose intensity is shaped cylindrically. It is a core component used to define regions of influence for terrain features, biome placement, and cave systems.

The class operates as a stateless function that maps a 3D coordinate to a scalar density value. Its primary architectural pattern is the composition of two one-dimensional functions—an axial curve for height and a radial curve for distance from the central Y-axis—to define a three-dimensional field. This design allows for the creation of highly complex and varied shapes (e.g., cones, hourglasses, bulging cylinders) by simply providing different curve functions, promoting extreme reusability.

Within the broader world generation system, CylinderDensity instances are nodes in a larger computational graph. The density values they produce are typically combined with outputs from other Density nodes through mathematical operations to form the final, complex density map of a world chunk.

### Lifecycle & Ownership
-   **Creation:** Instances are constructed by a higher-level world generator or a configuration system during the setup phase of a generation task. The critical `radialCurve` and `axialCurve` functions are injected via the constructor at this time.
-   **Scope:** The object is transient and its lifetime is scoped to a single, specific generation task, such as the generation of one world chunk. It holds no state between invocations.
-   **Destruction:** The instance is eligible for garbage collection as soon as the generation task that created it completes and releases its reference. There is no manual cleanup required.

## Internal State & Concurrency
-   **State:** **Immutable**. The object's behavior is defined entirely by the two `Double2DoubleFunction` instances provided at construction. These are stored in final fields and cannot be changed after instantiation. The class itself maintains no other internal state.
-   **Thread Safety:** **Fully Thread-Safe**. Due to its immutable nature, a single CylinderDensity instance can be safely invoked by multiple world generation threads concurrently without any need for external locking or synchronization.

    **Warning:** Thread safety is contingent on the injected `Double2DoubleFunction` implementations also being thread-safe and pure. Stateful or non-thread-safe functions will break this contract and lead to unpredictable generation results.

## API Surface
The public contract is minimal, consisting of the constructor for initialization and the core `process` method inherited from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context) | double | O(1) | Calculates the density value for the position specified in the context. This operation is a constant-time calculation. |

## Integration Patterns

### Standard Usage
CylinderDensity is not typically used in isolation. It is instantiated and composed by a larger generator which then invokes the `process` method for each point in a volume.

```java
// Example: Create a simple cone shape
// The radial curve decreases linearly with distance.
Double2DoubleFunction radialFalloff = (distance) -> Math.max(0.0, 1.0 - distance);

// The axial curve increases linearly with height.
Double2DoubleFunction axialRamp = (y) -> Math.max(0.0, y / 256.0);

// The generator constructs the node with its behavior.
CylinderDensity coneShape = new CylinderDensity(radialFalloff, axialRamp);

// A generator would then use it like this for a specific point.
Density.Context generationContext = new Density.Context(/* position, etc. */);
double density = coneShape.process(generationContext);
```

### Anti-Patterns (Do NOT do this)
-   **Null Injection:** Constructing a CylinderDensity with a null `radialCurve` or `axialCurve` will result in a `NullPointerException` during the `process` call. The `@Nonnull` annotation explicitly forbids this.
-   **Stateful Functions:** Injecting curve functions that rely on mutable external state is a severe anti-pattern. This breaks the immutability and thread-safety guarantees of the class, leading to non-deterministic and potentially buggy world generation. All injected functions should be pure.

## Data Pipeline
The data flow through this component is simple and direct. It acts as a single transformation step in a larger density calculation pipeline.

> Flow:
> World Generator -> Density.Context (with 3D Position) -> **CylinderDensity.process()** -> `double` (Density Value) -> Subsequent Aggregation or Voxel Material Selection

