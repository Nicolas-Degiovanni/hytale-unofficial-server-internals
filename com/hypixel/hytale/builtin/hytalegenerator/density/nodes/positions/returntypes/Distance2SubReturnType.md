---
description: Architectural reference for Distance2SubReturnType
---

# Distance2SubReturnType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.returntypes
**Type:** Transient

## Definition
```java
// Signature
public class Distance2SubReturnType extends ReturnType {
```

## Architecture & Concepts
The Distance2SubReturnType is a specialized calculation strategy component within the procedural world generation engine. It operates at the lowest level of the density function graph, which is responsible for defining the shape and substance of the game world.

Its primary role is to serve as a terminal operation for density nodes that compute values based on multiple distance fields. Specifically, it takes the results from two separate distance calculations, finds their difference, and normalizes this difference into a standardized range of -1.0 to 1.0.

This normalization is critical for the stability and predictability of the world generator. By mapping arbitrary distance differences to a consistent output range, the engine can reliably combine the output of this node with other density functions (e.g., Perlin noise, cellular noise) to create complex and coherent terrain features. It effectively transforms raw spatial information into a usable density value.

### Lifecycle & Ownership
- **Creation:** Instantiated by the world generator's configuration loader when parsing a density graph definition, typically from a data file like JSON. It is not a managed service or singleton.
- **Scope:** The lifetime of a Distance2SubReturnType instance is tied directly to the density graph node that owns it. It is a configuration object, not a long-lived runtime component.
- **Destruction:** The object is eligible for garbage collection as soon as the density graph configuration is fully processed and the generator is initialized, or when the world generator itself is discarded.

## Internal State & Concurrency
- **State:** Effectively immutable post-configuration. It inherits a single stateful field, maxDistance, from its parent. This field is set once during the initial parsing of the world generator's settings and is not modified thereafter. The get method is a pure function.
- **Thread Safety:** This class is inherently thread-safe. The get method contains no internal state, locks, or side effects. It operates solely on its input parameters and the immutable maxDistance field. It is designed to be called concurrently by multiple world generation worker threads without causing race conditions.

## API Surface
The public contract is minimal, consisting of a single computational method inherited from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(distance0, distance1, ...) | double | O(1) | Calculates the normalized difference between two input distances. Returns a value in the range [-1.0, 1.0]. Returns -1.0 if the primary closest point is null, and 0.0 if maxDistance is not a positive number. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is a component invoked by the internal machinery of a parent density node during world generation. The engine provides the necessary distance and position arguments during the evaluation of a specific point in world space.

```java
// Engine-level usage within a hypothetical Density Node
// Note: This class would not be instantiated directly in gameplay code.

// ... inside a density node's evaluation method ...
double d0 = calculateDistance0(samplePosition);
double d1 = calculateDistance1(samplePosition);
Vector3d p0 = getClosestPoint0(samplePosition);
Vector3d p1 = getClosestPoint1(samplePosition);

// The configured ReturnType instance is invoked
double finalDensity = this.returnType.get(d0, d1, samplePosition, p0, p1, context);

// ... use finalDensity for further calculations ...
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class manually using new Distance2SubReturnType(). Its creation and configuration are managed entirely by the data-driven world generator system.
- **Invalid Configuration:** Configuring a node to use this return type with a maxDistance of zero or less will neutralize its effect, causing the get method to always return 0.0. This can lead to unexpected flat or missing terrain features.

## Data Pipeline
This component acts as a transformation stage in the density calculation pipeline, converting raw distance metrics into a normalized density value.

> Flow:
> Density Node Evaluation -> Computes distance0 & distance1 -> **Distance2SubReturnType.get()** -> Normalized Density Value [-1.0, 1.0] -> Subsequent Aggregation or Modification

