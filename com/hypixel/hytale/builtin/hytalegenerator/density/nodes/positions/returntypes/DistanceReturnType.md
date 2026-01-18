---
description: Architectural reference for DistanceReturnType
---

# DistanceReturnType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.returntypes
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public class DistanceReturnType extends ReturnType {
```

## Architecture & Concepts
The DistanceReturnType class is a specialized component within Hytale's procedural world generation framework. It functions as a concrete **Strategy** for interpreting spatial data within a density-based generation system.

Its primary role is to translate a raw distance measurement—typically the output of a Signed Distance Field (SDF) or a similar geometric query—into a normalized density value. This normalized value, usually within the range of [-1.0, 1.0], is the fundamental unit used by subsequent stages of the world generator to determine material type, terrain shape, and solidity.

This class acts as a terminal transformation function in a density calculation graph. A series of generator nodes may define complex shapes, but it is this ReturnType that provides the final, standardized interpretation of a sample point's relationship to those shapes. The formula `distance / maxDistance * 2.0 - 1.0` is a classic technique to map a distance from [0, maxDistance] to the density range [-1, 1], where negative values typically signify "inside" a surface and positive values signify "outside".

## Lifecycle & Ownership
- **Creation:** Instances of DistanceReturnType are not created directly during the game loop. They are instantiated and configured by a higher-level system, such as a `DensityGraphLoader`, when parsing world generation presets from configuration files (e.g., JSON). The `maxDistance` field is set during this initial setup phase.
- **Scope:** The object's lifetime is bound to the world generator configuration it belongs to. It is created once when the generator is initialized and persists in memory as long as that generator is active.
- **Destruction:** The object is eligible for garbage collection when its parent world generator configuration is unloaded. This may occur during a server shutdown or when transitioning to a world with a different generation profile.

## Internal State & Concurrency
- **State:** The class is stateful, containing an inherited `maxDistance` field which dictates the normalization range. This state is intended to be configured once upon creation and treated as immutable thereafter. The `get` method itself is pure; it does not modify any internal state.
- **Thread Safety:** This class is **conditionally thread-safe**. The `get` method is re-entrant and can be safely invoked by multiple world generation worker threads simultaneously. However, this safety relies on the critical assumption that the `maxDistance` field is not mutated after its initial configuration. Any external modification to this field during generation would introduce a race condition and lead to non-deterministic and inconsistent terrain.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(distance0, distance1, samplePosition, closestPoint0, closestPoint1, context) | double | O(1) | Calculates a normalized density value from a raw distance. Returns 1.0 if the primary closest point is null, effectively treating it as infinitely far. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is uncommon. It is designed to be used polymorphically by the core density evaluation engine. The engine holds a reference to a `ReturnType` and invokes its `get` method without needing to know the concrete type.

```java
// Conceptual example from within the density graph evaluator
// The 'activeReturnType' is configured from a world preset and could be a DistanceReturnType instance.

ReturnType activeReturnType = worldPreset.getShapeReturnType();
double rawDistance = shapeNode.calculateDistance(samplePosition);

// The engine polymorphically calls get() to get the final density
double finalDensity = activeReturnType.get(rawDistance, ...);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DistanceReturnType()` in game logic. These objects are part of a declarative world generation system and must be managed by the configuration loader to ensure consistency.
- **Post-Initialization Mutation:** Modifying the `maxDistance` field after the world generator has begun processing chunks is unsafe. This will break the determinism of the generator and can cause visible seams or artifacts between chunks processed by different threads.

## Data Pipeline
This class is a key transformation step in the pipeline that converts a world coordinate into a final material decision.

> Flow:
> World Coordinate (Vector3d) -> Density Graph Node -> Raw Distance & Closest Point -> **DistanceReturnType.get()** -> Normalized Density (double) -> Terrain Mesher / Block Placer

