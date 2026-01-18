---
description: Architectural reference for ModelAsset
---

# ModelAsset

**Package:** com.hypixel.hytale.server.core.asset.type.model.config
**Type:** Asset Data Object

## Definition
```java
// Signature
public class ModelAsset implements JsonAssetWithMap<String, DefaultAssetMap<String, ModelAsset>> {
```

## Architecture & Concepts

The ModelAsset is a fundamental configuration object within the Hytale engine, acting as a comprehensive blueprint for any renderable and interactive entity. It is not merely a reference to a 3D model file; instead, it serves as a composite asset that aggregates numerous properties defining an entity's appearance, behavior, and physical presence in the world.

This class is a central component of the Hytale Asset System. Instances are not created procedurally at runtime but are deserialized from JSON files (e.g., `creature.json`) during the engine's initial asset loading phase.

The primary architectural pattern is declarative definition via a static **AssetBuilderCodec**. The `CODEC` field defines the complete schema for a ModelAsset, including:
-   **Data Mapping:** Which JSON keys map to which Java fields.
-   **Inheritance:** The `appendInherited` calls specify that if a property is missing, it should be inherited from a parent asset, enabling a powerful prefab-like system.
-   **Validation:** Each field is associated with validators (e.g., `CommonAssetValidator.MODEL_CHARACTER`, `Validators.nonNull`) that are executed at load time, ensuring data integrity and preventing runtime errors.
-   **Post-Processing:** The `afterDecode` hook allows for the transformation of loaded data into an optimized, runtime-ready format, such as converting a map of attachment weights into a `WeightedMap` for efficient random selection.

In essence, a ModelAsset is the single source of truth for an entity's static properties, queried by various engine systems like the renderer, physics engine, and animation controller to correctly instantiate and manage game objects.

## Lifecycle & Ownership

-   **Creation:** ModelAsset instances are created exclusively by the Hytale **AssetStore** during the server or client bootstrap sequence. The static `CODEC` is used to parse, validate, and construct the object from its corresponding JSON definition on disk. Direct instantiation is an anti-pattern and will result in an incomplete and non-functional object.
-   **Scope:** Once loaded, a ModelAsset is a shared, effectively immutable object that persists for the entire application session. All instances are stored in a static, globally accessible `AssetStore` managed by the `AssetRegistry`.
-   **Destruction:** Instances are not explicitly destroyed. They are garbage collected by the JVM when the application terminates and the `AssetRegistry` is unloaded.

## Internal State & Concurrency

-   **State:** The object's state is considered immutable after the `afterDecode` lifecycle hook completes. While fields are not marked as final, the governing pattern is that they are written to once during deserialization and are subsequently only read from. It caches both raw data from JSON (e.g., `randomAttachmentSets`) and derived, optimized data for runtime performance (e.g., `weightedRandomAttachmentSets`).

-   **Thread Safety:** The class is thread-safe for all read operations. Multiple game threads can safely access the same ModelAsset instance without external locking. Behavioral methods like `generateRandomScale` and `generateRandomAttachmentIds` use `ThreadLocalRandom` to ensure they can be called concurrently from different threads without contention, though they will produce non-deterministic results as designed.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | AssetStore | O(1) | **STATIC.** Retrieves the central repository for all loaded ModelAsset instances. |
| getBoundingBox() | Box | O(1) | Returns the non-null physics bounding box. Critical for collision detection. |
| generateRandomScale() | float | O(1) | Generates a random scale factor between `minScale` and `maxScale`. Used for entity variation. |
| generateRandomAttachmentIds() | Map | O(N) | Generates a map of selected attachment IDs based on configured weights. N is the number of attachment sets. |
| getAttachments(randomIds) | ModelAttachment[] | O(M+K) | Resolves and returns a final array of attachments, combining default attachments with the provided random selections. |

## Integration Patterns

### Standard Usage

A ModelAsset should always be retrieved from the central `AssetStore` via its unique string identifier. It is then used by systems to configure newly created entities.

```java
// How a developer should normally use this
AssetStore<String, ModelAsset, ?> store = ModelAsset.getAssetStore();
ModelAsset creatureAsset = store.get("hytale:boar");

if (creatureAsset != null) {
    Entity entity = world.spawnEntity();
    entity.setModel(creatureAsset);
    entity.setBoundingBox(creatureAsset.getBoundingBox());
    entity.setScale(creatureAsset.generateRandomScale());
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ModelAsset()`. This bypasses the asset loading pipeline, including deserialization, validation, inheritance, and post-processing. The resulting object will be malformed and cause `NullPointerException`s.

-   **Runtime State Mutation:** Do not modify the fields of a ModelAsset after it has been loaded. These assets are shared globally. Changing a property at runtime will affect every single entity using that asset, leading to unpredictable and difficult-to-debug visual and physical glitches.

-   **Direct Field Access to ASSET_STORE:** Avoid accessing the private static `ASSET_STORE` field. Always use the `getAssetStore()` static method, which handles lazy initialization and ensures the store is available.

## Data Pipeline

The flow of data from a configuration file on disk to a usable object in the game engine is managed entirely by the Asset System.

> Flow:
> `boar.json` on Disk -> AssetRegistry Scan -> **AssetBuilderCodec** (Deserialization & Validation) -> **ModelAsset** Instance -> Stored in `AssetStore` -> Entity Spawner requests `hytale:boar` -> Entity configured with ModelAsset properties

