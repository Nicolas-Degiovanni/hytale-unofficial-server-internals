---
description: Architectural reference for JsonAssetWithMap
---

# JsonAssetWithMap

**Package:** com.hypixel.hytale.assetstore.map
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface JsonAssetWithMap<K, M extends AssetMap<K, ?>> extends JsonAsset<K> {
}
```

## Architecture & Concepts
JsonAssetWithMap is a specialized contract interface that extends the base JsonAsset. It serves as a marker interface, signaling to the asset management system that a particular JSON asset is not a terminal data structure but is intrinsically linked to a collection of other assets, represented by an AssetMap.

This interface formalizes a composition pattern within the asset pipeline. By implementing JsonAssetWithMap, a class declares that its deserialized JSON data is designed to be a container or a manifest for a map of sub-assets. The asset loader uses this type information to trigger a secondary, recursive loading phase for the associated AssetMap, ensuring all dependent assets are resolved and linked before the parent asset is considered fully loaded.

It acts as a critical hint to the AssetStore, differentiating simple data assets (e.g., a configuration file) from complex composite assets (e.g., a definition for a set of themed UI elements, where each element is an entry in the map).

## Lifecycle & Ownership
As an interface, JsonAssetWithMap has no lifecycle of its own. It is a compile-time contract that does not exist as an object at runtime. The lifecycle—creation, scope, and destruction—is entirely determined by the concrete classes that implement it.

- **Creation:** Implementing classes are typically instantiated by the AssetStore's deserialization pipeline when a corresponding JSON file is loaded from disk.
- **Scope:** The scope of an implementing object is managed by the AssetStore's caching policy. It may persist for the duration of a game session or be evicted under memory pressure.
- **Destruction:** Instances are eligible for garbage collection when they are no longer referenced by the AssetStore cache or any other game system.

## Internal State & Concurrency
The interface itself is stateless.

- **State:** Any state is held by the implementing class. By contract, implementors are expected to contain or reference an AssetMap, making their state inherently mutable and complex.
- **Thread Safety:** This interface makes no guarantees about thread safety. Concurrency concerns are the responsibility of the implementing class. Systems accessing an object of this type should assume it is **not thread-safe** unless the specific implementation guarantees it. Access should be confined to the main game thread or synchronized externally.

## API Surface
JsonAssetWithMap introduces no new methods. It inherits its entire API surface from the parent JsonAsset interface. Its primary purpose is to provide a type marker for the asset system, not to add new behaviors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| (Inherited) | - | - | All methods are inherited from JsonAsset. Refer to JsonAsset documentation for its API contract. |

## Integration Patterns

### Standard Usage
The primary interaction pattern is type-checking within the asset loading system. A loader will deserialize a JSON file into a base object and then check if it conforms to the JsonAssetWithMap contract to determine the next processing step.

```java
// Example from within an asset loading service
JsonAsset<?> loadedAsset = deserialize(assetPath);

if (loadedAsset instanceof JsonAssetWithMap) {
    // This asset is a composite type. We must now locate and
    // load its associated AssetMap to fully resolve it.
    JsonAssetWithMap<?, ?> compositeAsset = (JsonAssetWithMap<?, ?>) loadedAsset;
    AssetMap<?, ?> associatedMap = resolveAndLoadMapFor(compositeAsset);
    // ... link the map to the parent asset
}
```

### Anti-Patterns (Do NOT do this)
- **Contract Violation:** Do not implement this interface on a class that does not logically own or correspond to an AssetMap. Doing so will mislead the asset system and may cause loading errors or unpredictable behavior.
- **Direct Casting:** Avoid casting a generic JsonAsset to JsonAssetWithMap without a prior `instanceof` check. This can lead to a ClassCastException if the asset is not a composite type.

## Data Pipeline
This interface alters the standard data flow for an asset, introducing a recursive step.

> Flow:
> Asset Request -> AssetStore Cache Miss -> File Read -> JSON Deserialization -> **Type Check for JsonAssetWithMap** -> [If True] Trigger Recursive Load of AssetMap -> Link Map to Parent Asset -> Populate Cache -> Return Completed Asset

