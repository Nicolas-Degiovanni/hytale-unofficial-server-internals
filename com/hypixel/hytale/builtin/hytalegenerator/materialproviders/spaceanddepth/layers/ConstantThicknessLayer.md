---
description: Architectural reference for ConstantThicknessLayer
---

# ConstantThicknessLayer<V>

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth.layers
**Type:** Value Object

## Definition
```java
// Signature
public class ConstantThicknessLayer<V> extends SpaceAndDepthMaterialProvider.Layer<V> {
```

## Architecture & Concepts
The ConstantThicknessLayer is a concrete implementation of the abstract **Layer** concept used by the **SpaceAndDepthMaterialProvider**. Its role within the world generation system is to define a horizontal stratum of material that has a uniform, non-varying thickness.

This class represents the simplest possible strategy for defining a layer. Unlike more complex implementations that might use noise functions or contextual data to vary their thickness, the ConstantThicknessLayer guarantees a fixed depth regardless of its position in the world. It serves as a fundamental building block for creating predictable geological formations, such as bedrock, topsoil, or simple floor and ceiling structures.

Architecturally, it is a leaf node in the composition of a world column's material profile. The parent **SpaceAndDepthMaterialProvider** aggregates a sequence of these Layer objects to build the final vertical slice of materials at any given coordinate.

## Lifecycle & Ownership
- **Creation:** Instances are created directly via their constructor, typically during the configuration phase of a biome or world feature definition. They are not managed by a dependency injection container or a central registry. The entity that defines a biome's material composition is responsible for instantiating it.
- **Scope:** The lifetime of a ConstantThicknessLayer instance is tied to its owning **SpaceAndDepthMaterialProvider**. It is a configuration object that persists as long as the generator profile is held in memory.
- **Destruction:** The object is eligible for garbage collection when its parent **SpaceAndDepthMaterialProvider** is dereferenced. It manages no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, consisting of the *thickness* and *materialProvider*, is established at construction and cannot be modified thereafter. Both fields are declared final.
- **Thread Safety:** This class is inherently **Thread-Safe**. Its immutability guarantees that multiple world generation threads can read its state concurrently without locks or any risk of data corruption. This is a critical feature for ensuring high performance in a parallelized world generation engine.

## API Surface
The public contract is minimal, focused entirely on providing the configured layer data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getThicknessAt(...) | int | O(1) | Returns the configured constant thickness. **Warning:** This method completely ignores all spatial and depth parameters passed to it. |
| getMaterialProvider() | MaterialProvider<V> | O(1) | Returns the material provider for this layer. This can be null, signifying a layer of empty space (e.g., air). |

## Integration Patterns

### Standard Usage
This class is intended to be instantiated and composed into a list or sequence that is then consumed by a **SpaceAndDepthMaterialProvider**.

```java
// Example: Defining a simple ground structure
// Assume 'stoneProvider' and 'dirtProvider' are valid MaterialProvider<V> instances.

// A 1-block thick layer of dirt on top
Layer<V> topsoil = new ConstantThicknessLayer<>(1, dirtProvider);

// A 5-block thick layer of stone underneath
Layer<V> rock = new ConstantThicknessLayer<>(5, stoneProvider);

// These layers would then be passed to a SpaceAndDepthMaterialProvider
// to define the material composition from the surface downward.
```

### Anti-Patterns (Do NOT do this)
- **Subclassing for Dynamic Behavior:** Do not extend this class to add logic for variable thickness. Its purpose is to be constant. If dynamic thickness is required, create a new, separate implementation of the **SpaceAndDepthMaterialProvider.Layer** interface.
- **Ignoring Null Material Provider:** The **getMaterialProvider** method can return null. Consumers must be robust and handle this case. A null provider typically implies that the layer should be composed of air or skipped entirely. Failure to check for null can result in a NullPointerException during world generation.

## Data Pipeline
The ConstantThicknessLayer does not process a stream of data. Instead, it acts as a static data source that is queried by the world generation pipeline.

> Flow:
> World Generator Engine -> SpaceAndDepthMaterialProvider -> **ConstantThicknessLayer.getThicknessAt()** -> Thickness (int)
>
> World Generator Engine -> SpaceAndDepthMaterialProvider -> **ConstantThicknessLayer.getMaterialProvider()** -> MaterialProvider<V> (or null)
>
> The returned values are then used by the **SpaceAndDepthMaterialProvider** to calculate the final material for a specific voxel in a world column.

