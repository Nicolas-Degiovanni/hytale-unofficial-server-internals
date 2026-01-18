---
description: Architectural reference for ContainedAssetCodec
---

# ContainedAssetCodec

**Package:** com.hypixel.hytale.assetstore.codec
**Type:** Utility

## Definition
```java
// Signature
public class ContainedAssetCodec<K, T extends JsonAssetWithMap<K, M>, M extends AssetMap<K, T>> implements Codec<K>, ValidatableCodec<K> {
```

## Architecture & Concepts

The ContainedAssetCodec is a specialized, high-performance codec designed to handle a critical engine pattern: **inline asset definitions**. It does not function as a traditional codec that fully deserializes an object from a data source. Instead, it acts as a deserialization interceptor.

Its primary role is to identify JSON objects that represent complete assets embedded within another asset's data file. Rather than fully hydrating these embedded objects immediately—which would be inefficient and disrupt the asset loading lifecycle—this codec performs a "deferred deserialization" process:

1.  **Identification:** It detects whether a field represents a reference to an existing asset (e.g., a string key) or an inline definition (a JSON object).
2.  **Extraction:** For inline definitions, it extracts the raw JSON text block without parsing its full structure.
3.  **Registration:** It wraps this raw data, along with a newly generated or inherited key, into a RawAsset object. This RawAsset is then registered with the parent asset's metadata for processing in a later stage of the asset pipeline.
4.  **Reference Injection:** It returns the key of the newly registered RawAsset, which is then stored in the parent object's field.

This mechanism effectively transforms an inline object definition into a standard asset reference at load time, ensuring that all assets, regardless of their source, are processed uniformly by the AssetStore. The behavior of key generation and parentage is controlled by the **Mode** enum, making this a highly configurable component for defining complex asset relationships.

### Lifecycle & Ownership

-   **Creation:** A ContainedAssetCodec is not instantiated at runtime for general use. It is constructed once during engine initialization as part of the static configuration for a specific AssetStore. It is typically defined alongside the schema for an asset that permits inline definitions.
-   **Scope:** The instance persists for the entire application session. It is a stateless utility that is registered with the engine's central codec system.
-   **Destruction:** The object is garbage collected upon final application shutdown when the asset system and its associated codec registries are torn down.

## Internal State & Concurrency

-   **State:** The ContainedAssetCodec is **immutable**. Its internal fields, including the delegate codec and operational mode, are final and set at construction. It holds no mutable state and does not cache data between calls.

-   **Thread Safety:** This class is inherently **thread-safe and re-entrant**. Its stateless and immutable nature ensures that it can be safely invoked by multiple asset loading threads simultaneously.

    **WARNING:** While the codec itself is thread-safe, it operates on and modifies the state of an AssetExtraInfo object passed to it. The caller is responsible for ensuring that the broader asset loading context, including the AssetRegistry and the AssetExtraInfo instance, is managed in a thread-safe manner.

## API Surface

The primary API consists of the standard Codec interface methods, which are invoked by the asset loading system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decodeJson(reader, extraInfo) | K | O(N) | Intercepts a JSON token stream. If a string, decodes as a key. If an object, extracts the raw JSON, registers it as a new RawAsset, and returns its key. N is the size of the inline JSON object. |
| decode(bsonValue, extraInfo) | K | O(N) | BSON equivalent of decodeJson. |
| encode(key, extraInfo) | BsonValue | O(M) | If the key represents a contained asset (conventionally prefixed with `*`), fetches the full asset and encodes it. Otherwise, encodes the key as a string reference. M is the size of the encoded asset. |
| toSchema(context) | Schema | O(1) | Generates a JSON schema definition that allows either a string reference or a full inline object definition. |
| validate(k, extraInfo) | void | O(1) | Defers validation of the referenced asset to the corresponding AssetStore. |

## Integration Patterns

### Standard Usage

This codec is never invoked directly by feature or game code. It is configured declaratively within an asset's schema definition to specify that a field can accept either a reference to an asset or an inline definition of that asset.

The following conceptual example shows how it might be registered for a `Weapon` asset that can define its `Projectile` inline.

```java
// Conceptual registration within the asset system
// This code does NOT appear in gameplay logic.

// 1. Get the codec for the asset type that can be contained (e.g., Projectile)
AssetCodec<HString, Projectile> projectileCodec = assetCodecs.get(Projectile.class);

// 2. Create a ContainedAssetCodec that wraps the projectile codec
//    Mode.INJECT_PARENT means the new projectile's parent will be the weapon.
ContainedAssetCodec<HString, Projectile, ProjectileMap> containedProjectileCodec =
    new ContainedAssetCodec<>(Projectile.class, projectileCodec, ContainedAssetCodec.Mode.INJECT_PARENT);

// 3. Register this codec for the 'projectile' field in the Weapon schema
weaponSchema.registerFieldCodec("projectile", containedProjectileCodec);

// RESULTING JSON USAGE:
// {
//   "name": "laser_rifle",
//   "projectile": {
//     "name": "laser_rifle_bolt", // This inline object is handled by ContainedAssetCodec
//     "speed": 500.0,
//     "damage": 25
//   }
// }
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ContainedAssetCodec()` in runtime logic. It is a configuration-time component of the asset system.
-   **Usage Outside AssetStore Context:** The `decode` methods will throw an `UnsupportedOperationException` if the provided `ExtraInfo` object is not an `AssetExtraInfo`. This codec is fundamentally coupled to the AssetStore pipeline.
-   **Incorrect Mode Selection:** Choosing the wrong `Mode` can lead to severe data integrity issues, such as assets with incorrect parents, orphaned assets, or key collisions. For example, using `GENERATE_ID` when a stable, predictable ID is required will break systems that rely on that asset's identity.

## Data Pipeline

The ContainedAssetCodec splits the data flow into two distinct stages: registration and processing.

**Stage 1: Registration (During Parent Asset Deserialization)**

> Flow:
> JSON File on Disk -> `RawJsonReader` -> Parent Asset Deserialization -> **ContainedAssetCodec.decodeJson** is invoked for an inline object -> A new `RawAsset` is created from the raw JSON text -> `RawAsset` is added to the parent's `AssetExtraInfo` metadata -> The new `RawAsset` key is returned and stored in the parent object's field.

**Stage 2: Processing (During AssetStore Finalization)**

> Flow:
> AssetStore finalization begins -> Iterates through all `RawAsset` instances collected during Stage 1 -> For each `RawAsset`, the standard `AssetCodec` (not the ContainedAssetCodec) is used to fully deserialize the raw data -> The fully hydrated asset object is placed into the `AssetMap`.

