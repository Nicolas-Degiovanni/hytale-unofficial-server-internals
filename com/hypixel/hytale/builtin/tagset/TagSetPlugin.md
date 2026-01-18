---
description: Architectural reference for TagSetPlugin
---

# TagSetPlugin

**Package:** com.hypixel.hytale.builtin.tagset
**Type:** Singleton

## Definition
```java
// Signature
public class TagSetPlugin extends JavaPlugin {
```

## Architecture & Concepts

The TagSetPlugin is a foundational server-side system responsible for managing and providing high-performance lookups for collections of tags, known as TagSets. It operates as a central registry for different categories of TagSets, such as NPC groups, item categories, or zone attributes.

Its primary architectural role is to bridge the gap between human-readable asset configuration files and the performance-critical needs of the game loop. It achieves this by ingesting asset data post-load, transforming it into a highly optimized, integer-indexed lookup structure, and exposing a simple, fast query interface.

The system is designed around two key components:
1.  **The Plugin:** The top-level TagSetPlugin class integrates with the server's plugin and asset loading lifecycle. It is responsible for registering TagSet asset types and managing the lifecycle of their corresponding lookup tables.
2.  **The Lookup:** The nested TagSetLookup class holds the final, "baked" data for a specific TagSet type. All runtime queries are directed to this class, which provides constant-time access to the processed tag data.

This separation ensures that the complex and potentially slow process of data ingestion and transformation happens only once during server initialization, while subsequent runtime queries are exceptionally fast.

### Lifecycle & Ownership
-   **Creation:** A single instance of TagSetPlugin is instantiated by the server's core plugin manager during the bootstrap sequence. Its lifecycle is directly managed by the server.
-   **Scope:** The singleton instance persists for the entire server session. The internal lookup tables it manages also persist for the same duration.
-   **Destruction:** The instance and all its associated data are marked for garbage collection during the server shutdown process when the plugin is disabled.

## Internal State & Concurrency
-   **State:** The internal state is highly mutable during the server's asset loading phase. The primary `lookups` map is populated as different systems register their TagSet types. Once the assets are loaded and processed via `putAssetSets`, the state of the inner `TagSetLookup` becomes immutable, enforced by an unmodifiable map wrapper. This transition from a write-heavy to a read-only state is critical to its design.

-   **Thread Safety:** The class is designed to be thread-safe for its intended use cases.
    -   Registration of new TagSet types is safe due to the use of a ConcurrentHashMap.
    -   Runtime queries via `tagInSet` are thread-safe because they operate on the immutable, pre-computed `flattenedSets` map.

    **Warning:** A race condition can occur if game logic attempts to query a TagSetLookup before the asset loading pipeline has completed and populated its data via `putAssetSets`. Systems must ensure they initialize after the asset loading stage is complete.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | TagSetPlugin | O(1) | Retrieves the global singleton instance of the plugin. |
| registerTagSetType(Class) | void | O(1) | Registers a new class of TagSet, preparing a lookup table for it. Must be called before assets of this type are loaded. |
| get(Class) | TagSetLookup | O(1) | Retrieves the specific lookup handler for a registered TagSet class. Throws NullPointerException if the class was never registered. |
| TagSetLookup.tagInSet(int, int) | boolean | O(1) | The primary query method. Performs a high-speed check to see if a tag index exists within a tag set index. |
| TagSetLookup.getSet(int) | IntSet | O(1) | Returns the complete, unmodifiable set of tag indices for a given tag set index. Returns null if the set does not exist. |

## Integration Patterns

### Standard Usage
The intended pattern is for a game system to retrieve the specific lookup table it needs during its own initialization, and then use it for frequent checks within its update logic.

```java
// During system initialization
TagSetPlugin.TagSetLookup npcGroupLookup = TagSetPlugin.get(NPCGroup.class);

// In performance-critical game logic (e.g., AI behavior tree)
int warriorGroupId = ... // get integer ID for "warriors"
int playerFactionTagId = ... // get integer ID for "player_faction"

if (npcGroupLookup.tagInSet(warriorGroupId, playerFactionTagId)) {
    // This NPC is part of the "warriors" group and has the "player_faction" tag.
    // Execute friendly logic.
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new TagSetPlugin()`. The server's plugin manager is the sole owner of its lifecycle. Always use the static `TagSetPlugin.get()` method.
-   **Premature Access:** Do not call `TagSetPlugin.get(MyType.class)` and attempt to use the lookup table before the corresponding assets for `MyType` have been loaded and processed. This will result in exceptions or incorrect data. Defer access until after the main asset loading phase.
-   **Repeated Lookups:** Do not repeatedly call `TagSetPlugin.get(NPCGroup.class)` inside a hot loop. Cache the returned `TagSetLookup` instance once during initialization.

## Data Pipeline
The plugin's primary function is to execute a data transformation pipeline that converts declarative configuration into a query-optimized data structure.

> Flow:
> JSON Asset File (e.g., `NPC/Groups/warriors.json`) -> HytaleAssetStore -> Deserialized `NPCGroup` objects -> **TagSetPlugin** (via `putAssetSets`) -> `TagSetLookupTable` processing -> Final `Int2ObjectMap<IntSet>` -> Game Logic Query via `tagInSet`

