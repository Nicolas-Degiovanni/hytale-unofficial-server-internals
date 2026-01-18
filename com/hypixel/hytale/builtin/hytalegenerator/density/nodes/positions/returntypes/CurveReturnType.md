---
description: Architectural reference for CurveReturnType
---

# CurveReturnType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.returntypes
**Type:** Strategy Object

## Definition
```java
// Signature
public class CurveReturnType extends ReturnType {
```

## Architecture & Concepts
The CurveReturnType is a specialized component within Hytale's procedural world generation framework, specifically operating within the Density Function graph. Its primary role is to act as a final transformation stage, applying a mathematical shaping function—a curve—to a pre-calculated distance value.

This class is a concrete implementation of the **Strategy** design pattern, with the abstract `ReturnType` class defining the contract. This decouples the core density calculation logic from the specific method used to interpret its results. By using a CurveReturnType, worldsmiths can precisely control the falloff, slope, and shape of terrain features without altering the more complex position and distance calculation nodes that precede it in the graph.

Crucially, this implementation is intentionally simplistic: it operates *only* on the first distance metric provided (`distance0`) and completely ignores all other spatial inputs, such as secondary distances or sample positions. This makes it a highly predictable and performant node for shaping density based on a single proximity value.

### Lifecycle & Ownership
- **Creation:** CurveReturnType instances are not created dynamically during gameplay. They are instantiated once during the server's bootstrap phase when the world generator's density graph is constructed from configuration files. A factory or builder responsible for parsing the generator definition is its creator.
- **Scope:** The object's lifetime is bound to the world generator instance it is a part of. It persists in memory for the entire duration that its parent generator is active.
- **Destruction:** It is marked for garbage collection when the world generator configuration is unloaded, for instance, upon server shutdown or when a world is deleted.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state consists of a single `final` field, `curve`, which is a `Double2DoubleFunction`. This function is injected via the constructor and cannot be changed after instantiation. The class itself holds no other mutable state.
- **Thread Safety:** This class is **inherently thread-safe**. Due to its immutable nature, a single instance can be safely shared and accessed by multiple world generation worker threads simultaneously without requiring any locks or synchronization primitives. This is critical for the performance of parallelized chunk generation.

## API Surface
The public contract is minimal, consisting only of the constructor and the overridden `get` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(distance0, ...) | double | O(C) | Calculates the final density value by applying the internal curve function to the `distance0` parameter. C is the complexity of the provided curve function, typically O(1). **Warning:** All other parameters are ignored. |

## Integration Patterns

### Standard Usage
A CurveReturnType is typically defined and instantiated as part of a larger density node configuration. It is not intended to be used as a standalone object.

```java
// Conceptual example of building a density node
// The actual implementation may use a configuration file (e.g., JSON/HOCON)

// 1. Define the shaping curve (e.g., a simple linear falloff)
Double2DoubleFunction linearCurve = (input) -> 1.0 - input;

// 2. Create the ReturnType strategy
ReturnType strategy = new CurveReturnType(linearCurve);

// 3. Inject it into a higher-level node that calculates distance
// This node would then call strategy.get(distance, ...) internally
PositionNode positionNode = new PositionNode(..., strategy);
```

### Anti-Patterns (Do NOT do this)
- **Relying on Ignored Parameters:** Do not configure a generator graph expecting this class to use `distance1`, `samplePosition`, or any parameter other than `distance0`. The implementation deliberately discards this data, and assuming otherwise will lead to generation logic that does not function as designed.
- **Null Injection:** The `curve` parameter is annotated as Nonnull. Passing a null function to the constructor will result in a `NullPointerException` during the `get` call, causing a catastrophic failure in the chunk generation pipeline for any chunk that uses this node.

## Data Pipeline
The CurveReturnType acts as a terminal processing step for a distance value within the larger density calculation pipeline.

> Flow:
> World Generator requests density at Vector3d -> Position Node calculates `distance0` -> **CurveReturnType.get(distance0)** -> The internal `curve` function is invoked -> The resulting double is returned -> The value is combined with other densities to produce the final terrain value.

