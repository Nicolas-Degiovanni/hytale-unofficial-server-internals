---
description: Architectural reference for MultiMixDensity
---

# MultiMixDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class MultiMixDensity extends Density {
```

## Architecture & Concepts
The MultiMixDensity class is a powerful compositing node within the procedural world generation's density function graph. Its primary role is to blend multiple, disparate Density functions into a single, continuous output. This blending is controlled by the output of a separate, single Density function, referred to as the *influence*.

Conceptually, MultiMixDensity acts as a multi-point gradient or a specialized 1D lookup table. Instead of mapping a value to a color or a simple number, it maps an influence value to a blend between two entire Density functions. This allows for the creation of complex and smooth transitions between different geological or biome features.

For example, an influence Density representing altitude could be used to mix a `WaterBedDensity` at low values, a `PlainsDensity` at medium values, and a `MountainPeakDensity` at high values. The MultiMixDensity node manages the linear interpolation between these functions based on the calculated altitude at any given point in the world.

The core components are:
-   **Influence Density:** A single Density whose output value (the influence) determines the blending factor.
-   **Keys:** A sorted list of points, where each Key maps a specific influence value to a corresponding Density function.
-   **Segments:** Internal representations of the space *between* two consecutive Keys. The system finds the correct segment for a given influence value and then interpolates the two associated Density functions.

## Lifecycle & Ownership
-   **Creation:** Instantiated during the construction of a world generator's density graph. This is typically driven by a higher-level configuration system that parses world generation presets. The constructor requires a list of Key objects and the initial influence Density.
-   **Scope:** The object's lifetime is bound to the density graph it belongs to. It persists as long as the world generator configuration is active and is discarded if the configuration is reloaded.
-   **Destruction:** Managed by the Java garbage collector. There are no native resources or explicit cleanup methods. It is reclaimed when the parent density graph is no longer referenced.

## Internal State & Concurrency
-   **State:** This class is stateful. It maintains a list of Segments and references to the various Density functions it manages. While most of its state is configured at creation and treated as immutable, the `influenceDensity` field can be modified post-construction via the `setInputs` method. This allows for dynamic rewiring of the density graph during initialization phases.
-   **Thread Safety:** This class is **not inherently thread-safe**. The `process` method is safe to call from multiple threads only if the underlying graph structure is not being modified simultaneously. The `setInputs` method presents a major concurrency hazard.

    **WARNING:** Modifying the inputs via `setInputs` while any thread is executing `process` will result in a race condition and undefined behavior. All graph wiring must be completed in a single-threaded context before initiating parallel world generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MultiMixDensity(List, Density) | constructor | O(N log N) | Constructs the node. Sorts the N keys and builds the internal segment list. Throws IllegalArgumentException for invalid key configurations. |
| process(Density.Context) | double | O(log N) | Computes the final density value. Performs a binary search on N segments to find the active pair, then processes up to two child densities. |
| setInputs(Density[]) | void | O(1) | Replaces the current influence Density. **Not thread-safe.** Expects an array with exactly one element. |

## Integration Patterns

### Standard Usage
MultiMixDensity is used to create strata or gradients in world generation. The developer defines a set of thresholds and associates a Density function with each, then provides a controlling Density to drive the transitions.

```java
// Define the density functions for different layers
Density oceanFloor = new NoiseDensity(...);
Density plains = new NoiseDensity(...);
Density mountains = new NoiseDensity(...);

// Define the controlling density, e.g., based on altitude
Density altitude = new YPositionDensity();

// Create keys mapping altitude values to densities
List<MultiMixDensity.Key> keys = List.of(
    new MultiMixDensity.Key(-32.0, oceanFloor),
    new MultiMixDensity.Key(0.0, plains),
    new MultiMixDensity.Key(64.0, mountains)
);

// Create the mixer node
MultiMixDensity terrainMixer = new MultiMixDensity(keys, altitude);

// This terrainMixer can now be used as a node in a larger graph
double finalValue = terrainMixer.process(context);
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never call `setInputs` on a MultiMixDensity instance that is being actively used by worker threads for generation. This will corrupt the state and lead to unpredictable results or crashes.
-   **Unsorted or Duplicate Keys:** While the constructor sorts the keys, providing keys with duplicate `value` fields will result in an `IllegalArgumentException`. The logic fundamentally relies on a unique, ordered set of thresholds.
-   **Insufficient Keys:** The constructor will throw an `IllegalArgumentException` if fewer than two keys are provided. A mix requires at least a starting and an ending point to define a segment.

## Data Pipeline
The flow of data for a single evaluation of this node is a multi-stage process involving several other Density nodes.

> Flow:
> 1. `Density.Context` (World Coordinates) is passed to `influenceDensity`.
> 2. `influenceDensity` returns a scalar `influence` value.
> 3. The `influence` value is used in a binary search to locate the correct `Segment` (the two keys that bracket the value).
> 4. `Density.Context` is passed to the two `Density` functions associated with the located segment's keys.
> 5. The two resulting density values are linearly interpolated based on the `influence` value's position within the segment.
> 6. The final blended `double` is returned.

