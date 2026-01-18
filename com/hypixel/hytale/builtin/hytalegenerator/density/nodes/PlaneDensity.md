---
description: Architectural reference for PlaneDensity
---

# PlaneDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class PlaneDensity extends Density {
```

## Architecture & Concepts
The PlaneDensity class is a foundational node within the procedural world generation's density graph system. It mathematically represents an infinite plane in 3D space, used to sculpt large-scale terrain features like ground levels, strata, or angled cliffs.

Its core function is to calculate a density value for any given point in the world. This value is not binary; it is determined by the point's perpendicular distance to the plane. The relationship between distance and the final density value is defined by a configurable **distanceCurve**, a functional interface that allows for highly flexible falloffs. This enables architects to define smooth gradients, sharp drop-offs, or other complex density transitions away from the plane's surface.

The class operates in two distinct modes, controlled by the **isAnchored** flag:
1.  **Unanchored Mode:** The plane is defined as passing through the world origin (0, 0, 0). Its orientation is determined solely by the **planeNormal** vector. This is suitable for defining global features like a sea level.
2.  **Anchored Mode:** The plane's position is relative to a dynamic anchor point provided by the incoming Density.Context. This allows the same plane definition to be reused to create features at different locations, such as floating islands or localized pockets of material.

An internal optimization, **isPlaneHorizontal**, is pre-calculated at construction to use faster math for planes aligned with the Y-axis, which is a common use case for terrain generation.

## Lifecycle & Ownership
-   **Creation:** A PlaneDensity object is instantiated by a higher-level system, typically a density graph parser or a procedural generation orchestrator. Its complete behavior (orientation, falloff curve, anchor mode) is defined at construction and is immutable thereafter.
-   **Scope:** The lifetime of a PlaneDensity instance is bound to the density graph it is a part of. These graphs are often assembled for a specific generation task, such as creating a world chunk column. As such, the object is typically short-lived.
-   **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It becomes eligible for collection as soon as the density graph that references it is no longer in use.

## Internal State & Concurrency
-   **State:** **Immutable**. All configuration fields—distanceCurve, planeNormal, and isAnchored—are final and set in the constructor. This design guarantees that the behavior of a PlaneDensity instance is consistent and predictable throughout its lifetime.
-   **Thread Safety:** **Fully thread-safe**. Because the class is immutable and its process method is a pure function of its inputs, a single PlaneDensity instance can be safely processed by multiple world-generation worker threads concurrently without any risk of race conditions or state corruption. No external locking is required.

## API Surface
The public contract is inherited from the base Density class and consists of a single processing method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(1) | Calculates the density value at the position specified in the context. Returns 0.0 if the plane normal is a zero vector or if an anchored plane is processed without a valid anchor in the context. |

## Integration Patterns

### Standard Usage
A PlaneDensity node is never used in isolation. It is constructed and composed with other Density nodes to form a complex procedural function.

```java
// Example: Creating a simple ground plane with a linear falloff.

// 1. Define the function that maps distance to density.
// Here, density is 1.0 at the plane and decreases linearly to 0.0 over 64 units.
Double2DoubleFunction linearFalloff = distance -> Math.max(0.0, 1.0 - distance / 64.0);

// 2. Define the plane's orientation (a flat, horizontal plane).
Vector3d groundPlaneNormal = new Vector3d(0.0, 1.0, 0.0);

// 3. Instantiate the node for a global, unanchored ground plane.
Density groundPlane = new PlaneDensity(linearFalloff, groundPlaneNormal, false);

// 4. In the generation loop, process it with a context.
Density.Context generationContext = new Density.Context(new Vector3d(150.0, 25.0, -400.0));
double densityAtPoint = groundPlane.process(generationContext);
```

### Anti-Patterns (Do NOT do this)
-   **External Vector Mutation:** Do not modify the Vector3d instance used for planeNormal after passing it to the constructor. The class does not create a defensive copy. Modifying the vector externally violates its immutability contract and will cause unpredictable generation artifacts, especially in a multithreaded environment.
-   **Zero-Vector Normal:** Avoid constructing a PlaneDensity with a zero-length normal vector. While the code gracefully handles this by returning 0.0, it represents a logical error in the density graph configuration and wastes processing cycles. The normal vector must have a non-zero length to define a valid plane.

## Data Pipeline
PlaneDensity acts as a transformation node within a larger data flow. It does not orchestrate a pipeline but is a critical step within one.

> Flow:
> Density.Context (contains world position & anchor) -> **PlaneDensity.process()** -> Distance Calculation (Vector Math) -> distanceCurve.get(distance) -> **double** (final density value)

