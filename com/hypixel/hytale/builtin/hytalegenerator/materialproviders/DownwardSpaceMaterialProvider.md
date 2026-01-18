---
description: Architectural reference for DownwardSpaceMaterialProvider
---

# DownwardSpaceMaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Transient Component

## Definition
```java
// Signature
public class DownwardSpaceMaterialProvider<V> extends MaterialProvider<V> {
```

## Architecture & Concepts
The DownwardSpaceMaterialProvider is a conditional filter component within the procedural world generation system. It functions as a **Decorator** for another MaterialProvider, wrapping it to add a single, specific constraint: the vertical distance from a ceiling.

Its primary role is to enable the creation of vertically layered materials in enclosed spaces like caves or under overhangs. For example, it can be used to ensure a specific ore only generates one block below a stone ceiling, or to create hanging features like glowing moss.

Architecturally, it is a fundamental building block in a **Chain of Responsibility** pattern. A world generator will typically query a composite list of MaterialProviders for a given coordinate. The DownwardSpaceMaterialProvider acts as a gate in this chain, returning a material only if its spatial condition is met. If the condition fails, it returns null, signaling the generator to continue querying the next provider in the chain.

### Lifecycle & Ownership
-   **Creation:** Instantiated during the configuration phase of world generation, typically when a biome or terrain feature definition is parsed and loaded. Its dependencies, the wrapped MaterialProvider and the integer space, are injected via its constructor.
-   **Scope:** The object's lifetime is bound to the world generator's active configuration. As it is stateless after construction, a single instance can be reused for the entire generation process of a world or region.
-   **Destruction:** Marked for garbage collection when the world generator's configuration is discarded, for instance, upon server shutdown or when a new world with a different ruleset is loaded.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal fields for the wrapped material provider and the target space are final and are set exclusively at construction time. This class does not modify its own state during its operational lifetime.
-   **Thread Safety:** **Thread-Safe**. Due to its immutable design, an instance can be safely accessed by multiple world generation worker threads concurrently without any external locking or synchronization. Its overall thread safety is contingent on the thread safety of the MaterialProvider it wraps.

## API Surface
The public contract is minimal, focused entirely on the implementation of the MaterialProvider interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelTypeAt(Context context) | V or null | O(1) + T(wrapped) | Returns a voxel type from the wrapped provider if and only if the context's *spaceBelowCeiling* value matches the configured space. Returns null otherwise. |

## Integration Patterns

### Standard Usage
This provider is designed to be composed with other providers to create highly specific placement rules. The standard pattern is to instantiate it as a wrapper around a more general provider.

```java
// 1. Define a base material provider, e.g., for a crystal voxel
MaterialProvider<Voxel> crystalProvider = new ConstantMaterialProvider<>(CRYSTAL_VOXEL);

// 2. Wrap it with DownwardSpaceMaterialProvider to restrict placement
//    to exactly 2 blocks below a ceiling.
MaterialProvider<Voxel> hangingCrystalProvider = new DownwardSpaceMaterialProvider<>(crystalProvider, 2);

// 3. The world generator uses the composed provider. It will only
//    return a CRYSTAL_VOXEL if the context's spaceBelowCeiling is 2.
Voxel type = hangingCrystalProvider.getVoxelTypeAt(generationContext);
```

### Anti-Patterns (Do NOT do this)
-   **Incorrect Chaining:** Placing this provider at the end of a generation chain after a generic, non-conditional provider (like a base stone provider) will cause it to never be evaluated, as the earlier provider will always return a material. It must be placed before more general fallbacks.
-   **Invalid Space Configuration:** Configuring the provider with a negative space value will ensure it never successfully provides a material, leading to silent failures in world generation logic.

## Data Pipeline
The DownwardSpaceMaterialProvider acts as a conditional filter in the material selection pipeline for a single voxel.

> Flow:
> World Generator -> CompositeMaterialProvider -> **DownwardSpaceMaterialProvider** (Checks Context.spaceBelowCeiling) -> [If Match] -> Wrapped MaterialProvider -> Voxel Type
>
> Flow (If No Match):
> World Generator -> CompositeMaterialProvider -> **DownwardSpaceMaterialProvider** -> Returns null -> CompositeMaterialProvider queries next provider in chain

