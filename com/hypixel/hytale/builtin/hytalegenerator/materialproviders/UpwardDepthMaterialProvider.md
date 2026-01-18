---
description: Architectural reference for UpwardDepthMaterialProvider
---

# UpwardDepthMaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Transient / Decorator

## Definition
```java
// Signature
public class UpwardDepthMaterialProvider<V> extends MaterialProvider<V> {
```

## Architecture & Concepts
The UpwardDepthMaterialProvider is a specialized component within the Hytale World Generation pipeline. It functions as a conditional filter, implementing the Decorator pattern to wrap another MaterialProvider instance.

Its core responsibility is to constrain the application of a nested material to a precise vertical depth. This depth is not an absolute world coordinate but a relative distance measured upwards from a surface, determined by the `depthIntoCeiling` property of the generation Context.

This mechanism is a fundamental building block for creating complex, stratified biomes and geological features. It enables designers to define material layers with high precision, such as ensuring a specific ore vein appears exactly two blocks deep into a cave ceiling or that a layer of topsoil is always one block thick. By composing these providers, intricate and predictable material structures can be achieved.

### Lifecycle & Ownership
- **Creation:** Instantiated during the configuration phase of world generation. It is typically constructed as part of a larger, composite MaterialProvider tree, often defined declaratively in world generation presets (e.g., JSON configuration files). It is not managed by a service locator or dependency injection container.
- **Scope:** Its lifetime is ephemeral, tied directly to the world generation task for a specific chunk or region.
- **Destruction:** The object is eligible for garbage collection as soon as the generation pass that utilizes it is complete. It holds no external resources and requires no explicit cleanup protocol.

## Internal State & Concurrency
- **State:** This object is **immutable**. Its internal fields, `materialProvider` and `depth`, are assigned only once during construction. It performs no caching and its behavior is entirely determined by its initial configuration and the input Context.
- **Thread Safety:** The UpwardDepthMaterialProvider is inherently **thread-safe**. Due to its immutable nature, a single instance can be safely shared and accessed by multiple world-generation worker threads without requiring locks or synchronization. This guarantee is contingent on the wrapped `materialProvider` also being thread-safe.

## API Surface
The public contract is minimal, focused exclusively on the overridden method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelTypeAt(Context context) | V (Nullable) | O(1) | Returns a voxel type from the nested provider only if the context's `depthIntoCeiling` matches the configured depth. Returns null otherwise. |

## Integration Patterns

### Standard Usage
This provider is not intended for standalone use. Its purpose is to be composed with other providers to add a depth-based constraint.

```java
// 1. Define a base material provider (e.g., for a specific type of stone)
MaterialProvider<VoxelType> graniteProvider = new VoxelTypeMaterialProvider<>(GRANITE);

// 2. Wrap the provider to restrict its application to a depth of 1 from a ceiling
MaterialProvider<VoxelType> ceilingGraniteLayer = new UpwardDepthMaterialProvider<>(graniteProvider, 1);

// 3. During world generation, the pipeline invokes the provider with the current context
// The result will be GRANITE only if context.depthIntoCeiling is 1.
VoxelType result = ceilingGraniteLayer.getVoxelTypeAt(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Misinterpreting Depth Coordinate:** Do not mistake the `depth` parameter for an absolute world Y-coordinate. It is a relative offset from a dynamically calculated surface. Attempting to use it to place materials at a fixed world height will lead to unpredictable and incorrect generation results.
- **Unnecessary Nesting:** Avoid wrapping a provider that already includes complex, multi-layered depth logic. This can create confusing, hard-to-debug behavior and redundant checks. Composition should be used to build logic, not to obscure it.

## Data Pipeline
The UpwardDepthMaterialProvider acts as a simple gate in the material selection data flow. It inspects context metadata and either terminates the chain for its branch or delegates to the next provider.

> Flow:
> World Generator -> **UpwardDepthMaterialProvider.getVoxelTypeAt(context)** -> [Depth Check: `context.depthIntoCeiling == this.depth`]
> - **Match:** -> Nested MaterialProvider.getVoxelTypeAt(context) -> Voxel Type
> - **No Match:** -> `null` (terminates this branch of material evaluation)

