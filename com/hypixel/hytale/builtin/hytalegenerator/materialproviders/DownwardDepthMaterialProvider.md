---
description: Architectural reference for DownwardDepthMaterialProvider
---

# DownwardDepthMaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Transient

## Definition
```java
// Signature
public class DownwardDepthMaterialProvider<V> extends MaterialProvider<V> {
```

## Architecture & Concepts
The DownwardDepthMaterialProvider is a specialized implementation of the MaterialProvider contract that functions as a conditional filter or gate. It embodies the **Decorator Pattern**, wrapping another MaterialProvider to modify its behavior without altering its class.

Its primary role within the world generation system is to restrict the placement of materials to a single, specific vertical depth. The generator queries the provider for a material at a given coordinate; this class first checks if the query's vertical depth (relative to a surface like a cave floor or terrain layer) matches its configured depth. If, and only if, the depths match, it delegates the query to the wrapped MaterialProvider. Otherwise, it returns null, effectively vetoing material placement at that location.

This component is a fundamental building block for creating stratified or layered geology. For example, a world designer could use it to ensure a thin layer of clay appears exactly two blocks below a riverbed, or that a specific ore vein only manifests at a precise depth from a cave floor.

### Lifecycle & Ownership
-   **Creation:** Instantiated declaratively within a world generation configuration, typically as part of a larger, composite provider structure. It is not managed by a dependency injection container and is constructed directly.
-   **Scope:** Short-lived. Its lifetime is bound to the execution of a specific world generation task, such as generating a single chunk. It holds no state that persists beyond this task.
-   **Destruction:** Becomes eligible for garbage collection as soon as the root generation process that created it completes.

## Internal State & Concurrency
-   **State:** **Immutable**. The delegate materialProvider and the target depth are final fields set at construction. An instance of this class cannot be modified after it is created.
-   **Thread Safety:** **Thread-safe**. Due to its immutable nature, a single instance can be safely shared and accessed by multiple world generation worker threads without external locking. Its overall thread safety is contingent on the thread safety of the wrapped MaterialProvider it delegates to.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelTypeAt(Context context) | V | O(1) | Returns a voxel type from the wrapped provider if the context depth matches the configured depth. Returns null otherwise. |

## Integration Patterns

### Standard Usage
This provider is almost always used to wrap another, more concrete provider, such as a ConstantMaterialProvider or a NoiseMaterialProvider, to constrain its effect to a specific depth.

```java
// This provider will only return the "GRAVEL" voxel type when the generator
// is processing the layer exactly 3 blocks below the current floor surface.
MaterialProvider<Voxel> gravelLayer = new ConstantMaterialProvider<>(GRAVEL);
MaterialProvider<Voxel> specificDepthLayer = new DownwardDepthMaterialProvider<>(gravelLayer, 3);

// The generator would then query specificDepthLayer...
Voxel type = specificDepthLayer.getVoxelTypeAt(generationContext);
```

### Anti-Patterns (Do NOT do this)
-   **Wrapping Null Providers:** The constructor is annotated as Nonnull. Passing a null delegate MaterialProvider will lead to a runtime exception or undefined behavior.
-   **Using Non-Positive Depth:** Configuring a depth of 0 or a negative number is syntactically valid but semantically nonsensical in most generation contexts, as depth is typically measured as a positive integer starting from 1. This will likely result in the provider never returning a material.
-   **Complex Layering:** For creating materials across a *range* of depths (e.g., from depth 2 to 5), do not chain multiple DownwardDepthMaterialProviders. This is inefficient and hard to maintain. Use a more appropriate component designed for ranges instead.

## Data Pipeline
The data flow is a simple conditional pass-through. The component acts as a gatekeeper for the generation context before it reaches the decorated provider.

> Flow:
> World Generator Query -> MaterialProvider.Context -> **DownwardDepthMaterialProvider** -> [Depth Check Failure] -> null
>
> World Generator Query -> MaterialProvider.Context -> **DownwardDepthMaterialProvider** -> [Depth Check Success] -> Wrapped MaterialProvider -> Voxel Type

