---
description: Architectural reference for CubeDensity
---

# CubeDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class CubeDensity extends Density {
```

## Architecture & Concepts
The CubeDensity class is a foundational node within the procedural world generation framework. It operates as a component in a larger density function graph, which collectively defines the shape of the world's terrain before it is materialized into blocks.

The primary purpose of this class is to generate a density field shaped like a cube centered at the origin (0,0,0). It achieves this by calculating the Chebyshev distance (the greatest distance along any coordinate axis) from the sample point to the origin. This distance value is then transformed by an injectable *falloff function*, which determines the final density.

This design employs a **Strategy Pattern** through the `falloffFunction`. CubeDensity is responsible for the geometric calculation (the "how" of measuring distance in a cube), while the caller provides the strategy for how that distance translates into a density value. This makes the component highly versatile for defining hard-edged cubes, cubes with soft falloff gradients, or other complex volumetric shapes based on a cubic metric.

## Lifecycle & Ownership
- **Creation:** Instances of CubeDensity are created and configured during the construction of a world generator's density graph. This is typically driven by a higher-level system that interprets world generation presets, often from configuration files. The critical `falloffFunction` dependency must be injected via the constructor.
- **Scope:** An instance lives for the duration of the world generator it is a part of. It is a stateless processing node whose configuration is immutable after creation.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup once the density graph that references it is no longer in use, for example, when a server shuts down or a different world generator is loaded.

## Internal State & Concurrency
- **State:** The class is **immutable**. Its only state is the `falloffFunction` field, which is marked as final and set during construction. It does not cache any results or modify its internal state during processing.
- **Thread Safety:** CubeDensity is inherently **thread-safe**. The `process` method is a pure function whose output depends only on its inputs. Multiple worker threads from the world generation engine can invoke the `process` method on a shared instance concurrently without any risk of race conditions or data corruption.

**WARNING:** While the CubeDensity class itself is thread-safe, the provided `falloffFunction` must also be thread-safe. Non-thread-safe or stateful lambda expressions can introduce concurrency bugs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CubeDensity(falloffFunction) | constructor | O(1) | Constructs a new density node with a specific falloff behavior. |
| process(context) | double | O(1) | Calculates the density for the position specified in the context. |

## Integration Patterns

### Standard Usage
CubeDensity is typically instantiated with a lambda or method reference for its falloff function and composed into a larger density graph.

```java
// Example: Create a cube with a linear falloff.
// Density is 1.0 at the center, decreasing to 0.0 at a distance of 32.
Double2DoubleFunction linearFalloff = distance -> Math.max(0.0, 1.0 - distance / 32.0);
Density cubeVolume = new CubeDensity(linearFalloff);

// In the engine, this would be called repeatedly by the density graph processor.
double densityValue = cubeVolume.process(new Density.Context(new Vector3d(10, 5, -15)));
```

### Anti-Patterns (Do NOT do this)
- **Stateful Falloff Function:** Avoid providing a `falloffFunction` that relies on or modifies external state. This breaks the assumption of thread safety and can lead to unpredictable generation results when processed by a multi-threaded engine.
- **Expensive Calculations:** The `process` method is in the hot path of world generation and is called millions of times. The provided `falloffFunction` must be computationally inexpensive. Complex operations like noise lookups, I/O, or heavy iteration should be avoided.

## Data Pipeline
CubeDensity acts as a transformation node within the world generation data pipeline. It does not manage a flow but rather participates in one.

> Flow:
> Density Graph Processor -> Provides **Density.Context** (with position) -> **CubeDensity.process** -> Calculates Chebyshev distance -> Invokes **falloffFunction** -> Returns **double** (density) -> Density Graph Processor

