---
description: Architectural reference for SolidityMaterialProvider
---

# SolidityMaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Transient

## Definition
```java
// Signature
public class SolidityMaterialProvider<V> extends MaterialProvider<V> {
```

## Architecture & Concepts
The SolidityMaterialProvider is a fundamental compositional component within the world generation pipeline. It does not generate material data itself; instead, it functions as a high-level conditional router, delegating the responsibility to one of two subordinate providers.

Its primary architectural role is to enforce a binary distinction between solid and non-solid space based on vertical position. This is the core mechanism for defining a world's surface, separating terrain that can be stood upon from the air or empty space above it.

This class embodies the Strategy and Decorator patterns. It encapsulates the logic for choosing a material generation strategy (solid vs. empty) and is composed at runtime with concrete provider implementations, such as a StoneMaterialProvider and an AirMaterialProvider.

## Lifecycle & Ownership
- **Creation:** Instantiated during the configuration phase of a world generator. It is constructed by higher-level generator logic that composes various providers to build a complete set of terrain generation rules. It is not managed by a central service registry.
- **Scope:** The lifetime of a SolidityMaterialProvider instance is typically scoped to a single world generation task or pass. It is a short-lived, configuration-based object.
- **Destruction:** The object is eligible for garbage collection as soon as the world generator configuration that references it goes out of scope. It manages no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** **Immutable**. The two delegate providers, solidMaterialProvider and emptyMaterialProvider, are final fields set during construction. The class contains no other mutable state. Its behavior is permanently fixed after instantiation.
- **Thread Safety:** **Thread-safe**. As an immutable object, it can be safely shared and accessed by multiple world generation worker threads without synchronization. Its overall thread safety is contingent on the thread safety of the providers it delegates to.

**WARNING:** While this class is inherently thread-safe, the providers it is configured with may not be. Ensure that any injected MaterialProvider instances are also immutable or properly synchronized.

## API Surface
The public contract is minimal, consisting of a single method inherited from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelTypeAt(Context context) | V | O(1) | Selects a delegate provider based on context.depthIntoFloor and requests the final voxel type. The complexity of the delegated call is not included. |

## Integration Patterns

### Standard Usage
This provider must be composed with two other concrete providers to function. The standard pattern involves injecting a provider for solid terrain and another for empty space.

```java
// 1. Define the concrete material providers
MaterialProvider<VoxelType> stoneProvider = new ConstantMaterialProvider<>(STONE_VOXEL);
MaterialProvider<VoxelType> airProvider = new ConstantMaterialProvider<>(AIR_VOXEL);

// 2. Compose them using SolidityMaterialProvider to define a surface
MaterialProvider<VoxelType> surfaceGenerator = new SolidityMaterialProvider<>(stoneProvider, airProvider);

// 3. The world generator uses the composed provider to get the final material
//    at a specific world coordinate and depth.
VoxelType result = surfaceGenerator.getVoxelTypeAt(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Incorrect Argument Order:** The constructor expects the solid provider first and the empty provider second. Reversing this order (`new SolidityMaterialProvider<>(airProvider, stoneProvider)`) will result in an inverted world where the ground is air and the sky is solid. This is a common and critical configuration error.
- **Recursive Composition:** Do not configure a SolidityMaterialProvider to delegate to itself, either directly or indirectly. This will cause a StackOverflowError during world generation.

## Data Pipeline
The SolidityMaterialProvider acts as a routing node within the larger material generation data flow. It receives a context object, inspects a single field, and forwards the request down one of two distinct branches.

> Flow:
> World Generator -> **SolidityMaterialProvider** -> (if depth > 0) -> SolidMaterialProvider -> Voxel Type
>
> World Generator -> **SolidityMaterialProvider** -> (if depth <= 0) -> EmptyMaterialProvider -> Voxel Type

