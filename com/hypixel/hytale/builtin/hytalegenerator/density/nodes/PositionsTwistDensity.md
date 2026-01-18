---
description: Architectural reference for PositionsTwistDensity
---

# PositionsTwistDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class PositionsTwistDensity extends Density {
```

## Architecture & Concepts
PositionsTwistDensity is a spatial deformation node within the procedural world generation engine. It functions as a decorator or filter that modifies the coordinate space before sampling a subsequent density function. Its primary purpose is to introduce localized, rotational warping into the terrain or density field.

This node operates by identifying a set of nearby control points, provided by a PositionProvider, and then twisting the sampling coordinate around those points. The magnitude and nature of the twist are governed by a configurable axis, a maximum distance of influence, and a Double2DoubleFunction curve that maps distance to a rotation angle.

Architecturally, this class is a key component for creating complex and organic geological features such as whirlpools, vortices, twisted caverns, or swirling magical biomes. It transforms a simple input density field into a more intricate one by applying a non-linear spatial warp. The final density value is determined by sampling the *input* density at the newly calculated, warped position. If multiple control points are within range, their influences are blended using a weighted average based on their distance to the sample point.

## Lifecycle & Ownership
-   **Creation:** Instances of PositionsTwistDensity are not typically created directly in game logic. They are instantiated during the initialization of the world generator, usually by a factory or builder class that deserializes a world generation configuration graph from a file. All dependencies and parameters are injected via the constructor.
-   **Scope:** The object's lifetime is bound to the world generator instance for a specific world or dimension. It persists as long as that generator is required for chunk generation.
-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup when the world generator it belongs to is discarded, for instance, during a server shutdown or when loading a different world.

## Internal State & Concurrency
-   **State:** This class is stateful. It maintains references to its input Density, a PositionProvider, and several configuration parameters like twistCurve, twistAxis, and maxDistance. Its state is configured at creation and is intended to be static during world generation, although it can be mutated via the setInputs method.

-   **Thread Safety:** The core process method is designed to be thread-safe and is frequently executed by multiple world generation worker threads concurrently. It achieves this by operating exclusively on local variables and the immutable state of the provided Density.Context. All intermediate data structures, such as lists of warp vectors and weights, are created on the stack for each call.

    **WARNING:** While the process method is safe for concurrent reads, the object itself is not fully immutable. Calling setInputs from one thread while another thread is executing process will lead to race conditions and undefined behavior. The density graph should be considered immutable after its initial construction.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PositionsTwistDensity(...) | constructor | O(1) | Constructs the node with its dependencies and configuration. Throws IllegalArgumentException for invalid maxDistance. |
| process(Density.Context) | double | O(N) | Calculates a warped sample position and returns the density from the input node at that new position. N is the number of positions returned by the PositionProvider within maxDistance. |
| setInputs(Density[]) | void | O(1) | Sets the upstream input density node. Used for graph assembly. **WARNING:** Not thread-safe. |

## Integration Patterns

### Standard Usage
This class is designed to be a node within a larger density graph. It wraps another Density node to apply the twist transformation before the wrapped node is evaluated.

```java
// Assume sourceDensity and positionSource are pre-configured
Density.Context context = new Density.Context(...);

// Configuration for a vertical twist
Vector3d verticalAxis = new Vector3d(0.0, 1.0, 0.0);
Double2DoubleFunction linearTwist = distance -> 180.0 * (1.0 - distance); // Twist 180 degrees at the center, 0 at maxDistance

PositionsTwistDensity twistNode = new PositionsTwistDensity(
    sourceDensity,
    positionSource,
    linearTwist,
    verticalAxis,
    64.0, // maxDistance
    true, // distanceNormalized
    false // zeroPositionsY
);

// The final value is from the sourceDensity, but sampled at a warped position
double densityValue = twistNode.process(context);
```

### Anti-Patterns (Do NOT do this)
-   **Excessive maxDistance:** Setting a very large maxDistance can severely degrade performance. The complexity of the process method is directly tied to the number of positions found within this radius. Keep the influence range reasonable.
-   **Concurrent Modification:** Do not call setInputs on a PositionsTwistDensity instance while it is being actively used for generation by worker threads. This will break thread safety guarantees.
-   **Null Input:** While the code handles a null input by returning 0.0, this typically indicates a configuration error in the density graph. The node becomes a no-op that returns a constant value, which is almost never the desired behavior.

## Data Pipeline
The data flow for a single density calculation involves transforming a coordinate, querying for influence points, and delegating to a child node.

> Flow:
> Density.Context (Original Position) -> **PositionsTwistDensity.process** -> PositionProvider query -> List of nearby positions -> Warp vectors calculated and blended -> Density.Context (Warped Position) -> Input Density.process -> double (Final Density Value)

