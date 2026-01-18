---
description: Architectural reference for BlockBoundingBoxes
---

# BlockBoundingBoxes

**Package:** com.hypixel.hytale.server.core.asset.type.blockhitbox
**Type:** Data Asset

## Definition
```java
// Signature
public class BlockBoundingBoxes implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, BlockBoundingBoxes>>, NetworkSerializable<Hitbox[]> {
```

## Architecture & Concepts

The BlockBoundingBoxes class is a server-side data asset that defines the physical collision geometry for a block type. It serves as a performance-critical component, bridging the gap between static asset definitions (JSON files) and the dynamic needs of the physics and gameplay systems.

Its primary architectural purpose is to **pre-compute and cache all possible rotational variants** of a block's hitbox. A block can be composed of one or more primitive Box shapes. Instead of performing expensive matrix transformations on these shapes every time a block's orientation is needed, this class calculates the geometry for every possible combination of yaw, pitch, and roll during the asset loading phase. At runtime, retrieving the correct rotated hitbox is a near-instantaneous array lookup.

This class is managed by the Hytale Asset System. The static CODEC field defines the deserialization logic, instructing the AssetStore how to parse a JSON file into a valid BlockBoundingBoxes instance. The class also implements NetworkSerializable, allowing its base geometry to be efficiently transmitted to clients, who can then perform the same rotational calculations.

Key architectural features include:
- **Pre-computation Cache:** The transient variants array holds all 64 (4x4x4) rotational outcomes, trading memory and load time for significant runtime performance gains.
- **Asset-Driven:** Instances are not created manually but are loaded from JSON files by the AssetStore, making block physics entirely data-driven.
- **Network Efficiency:** Provides a `toPacket` method to serialize its fundamental shape for network transmission.

## Lifecycle & Ownership

- **Creation:** Instances are created exclusively by the AssetStore during the server's asset loading bootstrap sequence. The static `CODEC` is invoked by the asset loader, which constructs an object and populates its fields from a corresponding JSON definition file. The `afterDecode` hook then calls `processConfig` to populate the rotational cache.
- **Scope:** An instance of BlockBoundingBoxes, once loaded, is considered a shared, immutable resource. It persists for the entire server session and is referenced by various game systems.
- **Destruction:** Instances are garbage collected when the AssetRegistry is shut down or reloaded, typically during a server stop or a full asset reload command.

## Internal State & Concurrency

- **State:** The object's state is **effectively immutable** after initialization. While its fields are not declared final, they are populated once during asset loading and are not intended to be modified thereafter. The `processConfig` method transitions the object from a raw data container to a fully processed, queryable state by populating the `variants` cache.
- **Thread Safety:** This class is **conditionally thread-safe**.
    - **Safe:** Once an instance is fully loaded and processed by the AssetStore, it is safe to read from multiple threads. All public accessor methods like `get` and `protrudesUnitBox` perform non-blocking reads on pre-computed data.
    - **Unsafe:** The static `getAssetStore` method uses lazy initialization for the `ASSET_STORE` field. Calling this method from multiple threads simultaneously before the store is initialized can lead to a race condition. Asset loading is expected to be a single-threaded, blocking operation during server startup, which mitigates this risk in practice.

**WARNING:** Do not modify the internal state of a BlockBoundingBoxes object after it has been loaded. Doing so will de-synchronize the pre-computed rotational variants and lead to unpredictable physics behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | AssetStore | O(1) | Static accessor for the global asset manager for this type. |
| get(Rotation yaw, Rotation pitch, Rotation roll) | RotatedVariantBoxes | O(1) | Retrieves the pre-computed hitbox variant for a specific orientation. This is the primary runtime accessor. |
| toPacket() | Hitbox[] | O(N) | Serializes the *base, unrotated* detail boxes into a network-ready format. N is the number of detail boxes. |
| protrudesUnitBox() | boolean | O(1) | Returns a cached boolean indicating if any rotational variant extends beyond the standard 1x1x1 block space. |

## Integration Patterns

### Standard Usage

BlockBoundingBoxes should always be retrieved from the central AssetStore using a known asset key. Game logic then queries for the specific rotational variant it needs for collision checks.

```java
// Retrieve the asset store for block bounding boxes
AssetStore<String, BlockBoundingBoxes, ?> store = BlockBoundingBoxes.getAssetStore();

// Get the bounding box definition for a specific block type, e.g., "Stairs"
BlockBoundingBoxes stairsHitbox = store.get("hytale:stairs");

// In the physics engine, get the pre-computed variant for a specific block's orientation
Rotation yaw = Rotation.Ninety;
Rotation pitch = Rotation.None;
Rotation roll = Rotation.None;
BlockBoundingBoxes.RotatedVariantBoxes variant = stairsHitbox.get(yaw, pitch, roll);

// Use the variant for collision detection
Box overallBounds = variant.getBoundingBox();
```

### Anti-Patterns (Do NOT do this)

- **Direct Instantiation:** Do not use `new BlockBoundingBoxes()`. This bypasses the asset loading system and the critical `processConfig` step. The resulting object will be in an invalid or incomplete state, lacking any rotational variants.
- **State Mutation:** Do not attempt to modify the `baseDetailBoxes` array or any other field on a loaded instance. This violates the immutability contract and will cause severe physics desynchronization.
- **Ignoring the Cache:** Do not manually re-implement the rotation logic found in the `rotate` methods. The entire purpose of this class is to perform this calculation once at load time. Always use the `get` method to retrieve a cached variant.

## Data Pipeline

The flow of data begins with a developer-defined JSON file and ends with a readily accessible, pre-computed geometry object used by the server's core physics systems.

> Flow:
> `hytale:stairs.json` -> Asset Loader -> **BlockBoundingBoxes.CODEC** -> `BlockBoundingBoxes` Instance -> `processConfig()` -> Populated `variants` Cache -> Physics Engine requests `get(yaw, pitch, roll)` -> Collision Check

