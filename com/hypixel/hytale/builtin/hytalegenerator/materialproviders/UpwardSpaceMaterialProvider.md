---
description: Architectural reference for UpwardSpaceMaterialProvider
---

# UpwardSpaceMaterialProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders
**Type:** Transient

## Definition
```java
// Signature
public class UpwardSpaceMaterialProvider<V> extends MaterialProvider<V> {
```

## Architecture & Concepts
The UpwardSpaceMaterialProvider is a conditional filter component within the world generation system. It implements the **Decorator** pattern, wrapping another MaterialProvider to add a specific spatial constraint before allowing it to execute.

Its primary function is to act as a guard based on the vertical clearance at a given coordinate. During procedural generation, the engine calculates the `spaceAboveFloor` for each potential voxel placement. This provider compares that runtime value against its configured `space` value. If and only if they match, it delegates the material request to the underlying, wrapped provider.

This mechanism is critical for creating realistic environments. For example, it prevents tall features like trees or large structures from generating inside caves or under overhangs where they would not physically fit. It is a fundamental building block for composing complex and context-aware generation rules.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly during the assembly of a world generation configuration, typically within a biome or feature definition. It is not managed by a dependency injection container or service registry.
-   **Scope:** The object's lifetime is bound to its parent configuration. It is a short-lived, stateless component used during a specific world generation pass and then discarded.
-   **Destruction:** The UpwardSpaceMaterialProvider is eligible for garbage collection as soon as the generation process that holds a reference to it completes. It requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** This object is **immutable**. Its internal fields, `materialProvider` and `space`, are set only once during construction and cannot be changed. It maintains no other internal state and performs no caching.
-   **Thread Safety:** The class is inherently **thread-safe**. Due to its immutable nature, a single instance can be safely shared and accessed by multiple world generation worker threads simultaneously without locks or synchronization, assuming the wrapped `materialProvider` is also thread-safe.

## API Surface
The public API consists of a single overridden method that forms its contract as a MaterialProvider.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelTypeAt(Context context) | @Nullable V | O(1) + T(wrapped) | Checks if `context.spaceAboveFloor` equals the configured `space`. If true, delegates to the wrapped provider. If false, returns null, effectively vetoing material placement. |

## Integration Patterns

### Standard Usage
This provider is intended to be chained with other providers to create a conditional rule. The developer wraps a base material provider to constrain its placement to locations with a specific amount of vertical open space.

```java
// Example: A provider that generates a large crystal structure
MaterialProvider<VoxelType> crystalProvider = new LargeCrystalProvider();

// Constrain the crystal to only generate in locations with exactly 15 blocks of vertical clearance
MaterialProvider<VoxelType> constrainedProvider = new UpwardSpaceMaterialProvider<>(crystalProvider, 15);

// The world generator now uses the constrained provider, which will return null
// in any location that does not have 15 blocks of space above it.
VoxelType voxel = constrainedProvider.getVoxelTypeAt(generationContext);
```

### Anti-Patterns (Do NOT do this)
-   **Incorrect Chaining:** Do not wrap this provider around a simple, ubiquitous provider (like a stone or dirt provider) unless the intent is to create hollow spaces. This is an inefficient way to achieve that result. It is designed for feature placement, not bulk terrain generation.
-   **Misconfiguration:** Configuring a `space` value of 0 or 1 may lead to unexpected behavior, as these values often represent solid ground or tight spaces where features are not intended to spawn.

## Data Pipeline
The UpwardSpaceMaterialProvider acts as a conditional gate in the data flow of material selection. It intercepts the request, inspects the context, and either forwards the request or terminates the chain for that location.

> Flow (Match):
> World Generator -> `getVoxelTypeAt(context)` -> **UpwardSpaceMaterialProvider** -> [Check `context.spaceAboveFloor` == `space`] -> Wrapped `materialProvider.getVoxelTypeAt(context)` -> VoxelType

> Flow (Mismatch):
> World Generator -> `getVoxelTypeAt(context)` -> **UpwardSpaceMaterialProvider** -> [Check `context.spaceAboveFloor` != `space`] -> `null`

