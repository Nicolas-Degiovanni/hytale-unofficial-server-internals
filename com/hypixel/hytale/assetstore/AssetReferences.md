---
description: Architectural reference for AssetReferences
---

# AssetReferences

**Package:** com.hypixel.hytale.assetstore
**Type:** Transient

## Definition
```java
// Signature
public class AssetReferences<CK, C extends JsonAssetWithMap<CK, ?>> {
```

## Architecture & Concepts
The AssetReferences class is a transient, command-like object that represents a pending dependency linkage between a set of parent assets and a new child asset. It is not a persistent entity but rather a short-lived handle used during the asset processing pipeline.

Its primary function is to encapsulate a group of parent assets, identified by their class and a set of keys, and provide a single method to establish a dependency relationship from all of them to a specified child asset. This pattern allows the system to batch-update asset dependency graphs, decoupling the identification of the parent assets from the action of linking them.

This class acts as a bridge between asset deserialization logic and the central AssetRegistry. When a shared dependency is discovered during asset loading, an AssetReferences instance is created to represent the group of assets that share this dependency. This instance is then used to perform the actual linkage against the global AssetStore.

## Lifecycle & Ownership
- **Creation:** Instantiated on-the-fly by asset loading or post-processing systems. It is typically created when a parser identifies a relationship that applies to multiple assets simultaneously.
- **Scope:** Method-scoped and extremely short-lived. An AssetReferences object is intended to be used immediately after creation and then discarded.
- **Destruction:** Eligible for garbage collection as soon as the `addChildAssetReferences` method call is complete and the instance falls out of scope. There is no explicit cleanup mechanism.

## Internal State & Concurrency
- **State:** Immutable. The internal state, consisting of the parent asset class and a set of parent keys, is defined at construction via `final` fields and cannot be modified thereafter.
- **Thread Safety:** The object itself is inherently thread-safe due to its immutability. However, its primary method, `addChildAssetReferences`, mutates shared global state by interacting with the AssetRegistry and the corresponding AssetStore.

**WARNING:** While an instance of AssetReferences can be safely passed between threads, calls to `addChildAssetReferences` are not guaranteed to be thread-safe. The safety of this operation is entirely dependent on the concurrency guarantees of the underlying AssetRegistry and AssetStore implementations. Concurrent modifications to the asset graph without external synchronization can lead to race conditions and data corruption.

## API Surface
The public contract is minimal, focusing on a single action.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AssetReferences(Class, Set) | constructor | O(1) | Creates a handle for a group of parent assets. |
| addChildAssetReferences(Class, K) | void | O(N) | Establishes a dependency from all parent assets to the specified child. N is the number of parent keys. This method modifies global state. |
| getParentAssetClass() | Class | O(1) | Returns the class of the parent assets. |
| getParentKeys() | Set | O(1) | Returns the set of keys identifying the parent assets. |

## Integration Patterns

### Standard Usage
The intended pattern is to create an instance, immediately use it to link a child asset, and then discard it. This is common within asset post-processing logic.

```java
// Assume 'stoneVariants' is a Set<BlockKey> for all stone blocks
// and 'StoneBlockAsset.class' is the parent asset type.
AssetReferences<BlockKey, StoneBlockAsset> stoneAssetRefs = new AssetReferences<>(StoneBlockAsset.class, stoneVariants);

// Link all stone block variants to a single particle effect asset
stoneAssetRefs.addChildAssetReferences(ParticleEffectAsset.class, "hytale:stone_break_particles");
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store instances of AssetReferences in fields or collections. They represent a transient operation, not persistent state. Storing them is a memory leak and conceptually incorrect.
- **Modification after Creation:** The object is immutable. Do not attempt to use reflection or other means to modify its internal key set after construction.
- **Unsafe Concurrent Writes:** Do not call `addChildAssetReferences` from multiple threads without ensuring that the target AssetStore is thread-safe or that proper locking is implemented around the calls.

## Data Pipeline
AssetReferences is not a step in a data flow pipeline; rather, it is an actor that initiates a state change in a global system.

> Flow:
> Asset Deserializer -> Identifies shared dependency -> **new AssetReferences(...)** -> **.addChildAssetReferences(...)** -> AssetRegistry -> AssetStore Update

