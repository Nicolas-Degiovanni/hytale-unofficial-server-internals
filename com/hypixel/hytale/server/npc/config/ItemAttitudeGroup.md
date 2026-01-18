---
description: Architectural reference for ItemAttitudeGroup
---

# ItemAttitudeGroup

**Package:** com.hypixel.hytale.server.npc.config
**Type:** Data Asset

## Definition
```java
// Signature
public class ItemAttitudeGroup implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, ItemAttitudeGroup>> {
```

## Architecture & Concepts
The ItemAttitudeGroup class is a data-driven configuration asset, not a service or manager. Its primary function is to define a set of NPC behavioral responses towards specific items, which are identified by tags. This allows designers to create reusable "attitude profiles" (e.g., "hates magical items", "loves shiny things") that can be applied to various NPC types.

Architecturally, this class serves as a critical data translation layer between human-readable configuration files (JSON) and the core server-side AI systems. It consumes a simplified `Sentiment` enum from the asset files and transforms it into the engine's canonical `Attitude` enum during an `afterDecode` hydration step. This decouples game data from engine code, allowing designers to work with intuitive terms like "Like" or "Dislike" while the engine operates on a more rigid set of states like "FRIENDLY" or "HOSTILE".

All instances of this class are loaded and managed centrally by the AssetRegistry. Access is provided through a static, lazily-initialized `IndexedLookupTableAssetMap`, ensuring efficient, global lookup of any defined attitude group by its string identifier.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Asset System during the server's asset loading phase. The static `CODEC` field dictates the precise deserialization and object hydration logic, including the critical transformation of `sentiments` to `attitudes`. Direct instantiation by developers is strictly forbidden.
- **Scope:** The lifecycle of an ItemAttitudeGroup instance is bound to the AssetRegistry. It is loaded once when the server starts or reloads assets and persists in memory for the duration of the server session.
- **Destruction:** Objects are marked for garbage collection only when the AssetRegistry is cleared or re-initialized, typically during a full server shutdown or a comprehensive asset reload. There are no manual destruction or cleanup methods.

## Internal State & Concurrency
- **State:** An ItemAttitudeGroup instance is effectively immutable after its creation and hydration by the asset loader. The initial state is read from a JSON asset into the `sentiments` map, which is then processed into the final, engine-ready `attitudes` map. This final map is the only state exposed through the public API.
- **Thread Safety:** The object is thread-safe for all read operations after it has been fully constructed by the asset system. The static `getAssetMap` method, which provides access to the central cache, is not internally synchronized.

**WARNING:** The initial population of the static `ASSET_MAP` cache is not protected by a lock. It is assumed that the first call to `getAssetMap` will occur in a controlled, single-threaded context, such as the main server initialization thread. Concurrent first-time access from multiple threads will lead to a race condition.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Retrieves the global, cached map of all loaded ItemAttitudeGroup assets. |
| getId() | String | O(1) | Returns the unique string identifier for this asset group. |
| getAttitudes() | Map<Attitude, String[]> | O(1) | Returns the fully processed map of engine attitudes to their corresponding item tags. |

## Integration Patterns

### Standard Usage
The intended pattern is to retrieve the global asset map once and then perform lookups for specific attitude groups as needed by the NPC AI or behavior systems.

```java
// Retrieve the central map of all attitude groups
IndexedLookupTableAssetMap<String, ItemAttitudeGroup> allGroups = ItemAttitudeGroup.getAssetMap();

// Fetch a specific group by its ID (e.g., "undead_item_preferences")
ItemAttitudeGroup undeadAttitudes = allGroups.get("undead_item_preferences");

if (undeadAttitudes != null) {
    // Access the processed attitude data for use in AI logic
    Map<Attitude, String[]> attitudeMap = undeadAttitudes.getAttitudes();
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ItemAttitudeGroup()`. This bypasses the asset loading system and results in an empty, un-hydrated object that is not registered in the global asset map. Such objects will cause NullPointerExceptions and incorrect AI behavior.
- **State Modification:** Do not attempt to modify the map returned by `getAttitudes()`. While the underlying implementation may be mutable, the contract assumes read-only access. Modifying this shared state will lead to unpredictable behavior across all NPCs using this profile.
- **Early Access:** Do not call `getAssetMap` from a system that may initialize before the AssetRegistry is fully populated. This can result in an empty or incomplete map.

## Data Pipeline
The flow of data for this component begins with a raw asset file and ends with a hydrated, engine-ready object used by the server's AI systems.

> Flow:
> `item_attitude_group.json` -> `AssetRegistry` (Deserialization) -> **`ItemAttitudeGroup.CODEC`** (Hydration & Transformation) -> Instance in `ASSET_MAP` -> `NPC Behavior System`

