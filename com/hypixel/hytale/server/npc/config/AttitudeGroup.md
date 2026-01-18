---
description: Architectural reference for AttitudeGroup
---

# AttitudeGroup

**Package:** com.hypixel.hytale.server.npc.config
**Type:** Data Model

## Definition
```java
// Signature
public class AttitudeGroup implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, AttitudeGroup>> {
```

## Architecture & Concepts
The AttitudeGroup class is a data-driven configuration model, not an active service. Its primary role is to represent the social disposition of one group of NPCs towards other groups. It encapsulates the rules that determine whether an NPC faction is hostile, friendly, neutral, or fearful of another.

This class is a direct representation of a JSON asset file on disk. It is designed to be deserialized and managed exclusively by the Hytale Asset System. The static `CODEC` field is the core of this mechanism; it is a declarative blueprint that instructs the `AssetRegistry` how to parse the corresponding JSON and populate an AttitudeGroup instance.

All loaded AttitudeGroup instances are stored in a globally accessible, lazily-initialized static map, `ASSET_MAP`. This provides a centralized, performant registry for the server's AI and behavior systems to query NPC relationship data without needing to manage object lifecycles or pass references. This class acts as the in-memory schema for NPC faction configuration.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the `AssetRegistry` during the server's asset loading phase at startup. The `AssetBuilderCodec` defined in the static `CODEC` field is invoked to construct and populate each object from its corresponding JSON asset file. **Manual instantiation is strictly forbidden.**
- **Scope:** Application-scoped. Once loaded by the asset system, AttitudeGroup instances are considered immutable configuration constants that persist for the entire server session.
- **Destruction:** Instances are garbage collected when the server shuts down and the `AssetRegistry` is cleared. There is no mechanism for manual destruction or hot-reloading of individual groups.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal fields, including the `id` and the `attitudeGroups` map, are populated once during deserialization. While the `attitudeGroups` map is not technically a `java.util.concurrent` collection, it is never mutated after initialization and should be treated as read-only.
- **Thread Safety:** The object is thread-safe for read operations due to its immutable-after-initialization nature. The static `getAssetMap` method contains a lazy-initialization block. While asset loading is typically a single-threaded bootstrap process, calling this method from multiple threads *before* the `AssetRegistry` is fully initialized could theoretically lead to a race condition. However, under standard engine operation, this is not a practical concern.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Retrieves the global, read-only map of all loaded AttitudeGroup assets. The first call performs a lookup in the AssetRegistry. |
| getId() | String | O(1) | Returns the unique identifier for this attitude group, corresponding to the asset name. |
| getAttitudeGroups() | Map | O(1) | Returns the map defining attitudes towards other NPC groups. The keys are Attitude enums and values are arrays of group IDs. |

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve the global asset map once and then query it for specific attitude configurations as needed. This is typically done within higher-level AI or NPC behavior systems.

```java
// Retrieve the central map of all attitude groups
IndexedLookupTableAssetMap<String, AttitudeGroup> allGroups = AttitudeGroup.getAssetMap();

// Look up a specific group by its asset ID (e.g., "undead")
AttitudeGroup undeadAttitudes = allGroups.get("undead");

if (undeadAttitudes != null) {
    // Check which groups the "undead" are hostile towards
    String[] hostileTargets = undeadAttitudes.getAttitudeGroups().get(Attitude.HOSTILE);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new AttitudeGroup()`. This bypasses the asset system, creating an unmanaged and uninitialized object that will not be present in the global `ASSET_MAP` and will cause `NullPointerException`s in systems that expect fully-populated objects.
- **State Mutation:** Do not attempt to modify the map returned by `getAttitudeGroups`. This is shared, global configuration data. Modifying it at runtime will lead to unpredictable and non-deterministic AI behavior across the entire server.
- **Premature Access:** Do not call `getAssetMap` before the server's `AssetRegistry` has completed its loading phase. This will result in a failed lookup and likely cause a server crash.

## Data Pipeline
The AttitudeGroup class is populated at the end of a data loading pipeline. It does not process streaming data itself; rather, it is the result of that process.

> Flow:
> JSON Asset File (`/assets/.../attitudegroup/undead.json`) -> Server Asset Loader -> **AttitudeGroup.CODEC** -> Populated `AttitudeGroup` Instance -> Stored in `AttitudeGroup.ASSET_MAP` -> Queried by NPC Behavior System

