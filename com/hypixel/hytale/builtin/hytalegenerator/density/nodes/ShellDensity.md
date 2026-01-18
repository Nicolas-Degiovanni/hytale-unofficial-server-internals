---
description: Architectural reference for ShellDensity
---

# ShellDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class ShellDensity extends Density {
```

## Architecture & Concepts
ShellDensity is a foundational component within the procedural world generation framework, specifically operating as a node in the density function pipeline. Its primary role is to generate a scalar density field shaped like a shell or dome, centered at the origin.

This class transforms a 3D spatial coordinate into a single density value. The transformation is governed by two key geometric properties:
1.  **Distance:** The point's distance from the origin (0,0,0).
2.  **Angle:** The angle between the point's position vector and a predefined axis.

These properties are mapped through user-provided function curves, `distanceCurve` and `angleCurve`, allowing for highly art-directable shapes. For example, it can be used to generate the cap of a mushroom, a hollow sphere, or a directed cone of influence. Its stateless, input-driven design makes it a highly reusable and composable primitive for more complex procedural features.

## Lifecycle & Ownership
-   **Creation:** ShellDensity is instantiated by higher-level world generation logic, typically a feature generator or a configuration system that assembles a graph of density functions. It is not a managed service and should not be treated as one.
-   **Scope:** The lifetime of a ShellDensity instance is typically short and tied to a specific, localized generation task, such as generating a single world feature or processing one chunk.
-   **Destruction:** The object is eligible for garbage collection as soon as the generation task that created it completes and discards its reference. There is no manual cleanup required.

## Internal State & Concurrency
-   **State:** The internal state of ShellDensity is defined at construction and is **effectively immutable**. The configuration fields (`angleCurve`, `distanceCurve`, `axis`, `isMirrored`) are final. This design ensures that a configured ShellDensity instance is a pure, deterministic function.

-   **Thread Safety:** This class is **fully thread-safe**. The `process` method is a pure function with no side effects. It only reads from its immutable configuration and the provided `Density.Context`. This is a critical architectural feature that allows the world generator to process density fields in parallel across multiple threads without locking or synchronization.

    **WARNING:** While the reference to the `axis` Vector3d is final, the Vector3d class itself may be mutable. Modifying the `axis` vector externally after it has been passed to the constructor will lead to unpredictable and non-deterministic behavior. See Anti-Patterns.

## API Surface
The public contract is minimal, consisting of the constructor for configuration and the `process` method for execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ShellDensity(angleCurve, distanceCurve, axis, isMirrored) | constructor | O(1) | Constructs a new density node with the specified shaping parameters. |
| process(context) | double | O(1) | Calculates the density at the position specified in the context. Returns 0.0 if the axis has zero length. |

## Integration Patterns

### Standard Usage
A ShellDensity node is typically instantiated as part of a larger composite density function. The caller configures it with curves and an axis to define a specific shape, then repeatedly calls `process` for various points in a volume.

```java
// Example: Creating a simple dome shape
Vector3d verticalAxis = new Vector3d(0, 1, 0);
Double2DoubleFunction distanceFalloff = d -> Math.max(0, 1.0 - d / 10.0); // Linear falloff over 10 units
Double2DoubleFunction angleShape = a -> Math.cos(Math.toRadians(a)); // Strongest at the pole (0 deg), zero at equator (90 deg)

ShellDensity domeGenerator = new ShellDensity(angleShape, distanceFalloff, verticalAxis, false);

// In the generation loop for a given point 'pos'
Density.Context ctx = new Density.Context(pos);
double density = domeGenerator.process(ctx);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Do not modify the `axis` vector after passing it to the constructor. This breaks the immutability contract and will cause severe, hard-to-debug issues in a multithreaded environment. Always pass a new or cloned vector if its source is not guaranteed to be constant.
-   **Zero-Length Axis:** Providing a zero-length vector for the `axis` is computationally wasteful. The `process` method will short-circuit and return 0.0, but the check is performed on every call. Configurations should validate that the axis has a non-zero length before instantiation.
-   **Direct Instantiation in Performance Loops:** Avoid creating new ShellDensity instances inside a tight generation loop. Instantiate it once per feature and reuse it for all points within that feature's volume.

## Data Pipeline
The data flow for this component is a direct, stateless transformation. It is a single stage in a potentially much larger density calculation pipeline.

> Flow:
> Generator provides `Density.Context` -> **ShellDensity** calculates distance & angle -> Maps values via internal curves -> Returns final `double` density -> Subsequent stage (e.g., Combiner Node, Mesher) consumes density value

