---
description: Architectural reference for TagSetLookupTable
---

# TagSetLookupTable<T extends TagSet>

**Package:** com.hypixel.hytale.builtin.tagset
**Type:** Utility

## Definition
```java
// Signature
public class TagSetLookupTable<T extends TagSet> {
```

## Architecture & Concepts
The TagSetLookupTable serves as a specialized compiler and resolver for the Hytale Tagging system. Its primary architectural role is to transform a declarative, hierarchical graph of TagSet definitions into a highly optimized, "flattened" lookup table. This process is a critical pre-computation step performed during the engine's loading phase.

The system takes human-readable TagSet configurations—which may include nested references, inclusions, exclusions, and glob-based wildcards—and resolves them into a simple map of integer-to-integer-set. Each key in the final map represents a TagSet's unique integer ID, and the value is a complete set of all tag IDs that belong to it after the entire hierarchy has been resolved.

Key architectural features include:
*   **Hierarchy Resolution:** It recursively processes `includedTagSets` and `excludedTagSets` to build a final, consolidated set of tags.
*   **Glob Pattern Matching:** It supports wildcard patterns (e.g., `hytale:wood_*`) for including or excluding large groups of tags efficiently.
*   **Cyclic Dependency Detection:** The resolver actively tracks its traversal path to detect and prevent infinite loops caused by circular references between TagSets, throwing an IllegalStateException to fail the loading process cleanly.

By performing this expensive resolution once, the TagSetLookupTable enables other engine systems to perform complex tag queries at runtime with near O(1) complexity, simply by checking for the presence of an integer in a hash set.

## Lifecycle & Ownership
-   **Creation:** This object is instantiated by a higher-level management system, such as a TagSetPlugin or AssetManager, during a data loading or world initialization phase. It is created only after all raw TagSet configuration files have been parsed and their corresponding string-to-integer index maps have been generated.
-   **Scope:** A TagSetLookupTable is a transient object, but the data it produces is long-lived. The instance itself is typically discarded after its constructor completes and the resulting flattened map is retrieved. The generated map is then cached by the management system and persists for the duration of a game session or until a full resource reload is triggered.
-   **Destruction:** The object is eligible for garbage collection immediately after its resulting `tagMatcher` map is extracted and stored by its owner.

## Internal State & Concurrency
-   **State:** The primary internal state is the `tagMatcher` map, which is progressively built during the object's construction. After the constructor finishes, the object's state is effectively immutable. It performs a complex, stateful graph traversal during initialization but exposes only the final, read-only result.
-   **Thread Safety:** This class is **not thread-safe** and must be confined to a single thread during construction. The internal algorithms for cycle detection rely on a thread-local `path` list. Once constructed, the resulting `Int2ObjectMap` returned by `getFlattenedSet` is safe for concurrent reads by multiple threads, provided no external code attempts to modify the returned collections.

## API Surface
The public contract is minimal, focusing entirely on the creation and retrieval of the resolved data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TagSetLookupTable(tagSetMap, tagSetIndexMap, tagIndexMap) | constructor | O(V+E) | Constructs and resolves the entire TagSet graph. V is the number of TagSets, E is the total number of include/exclude relationships. Throws IllegalStateException on cyclic dependency. |
| getFlattenedSet() | Int2ObjectMap<IntSet> | O(1) | Returns the final, pre-computed lookup table. This is the primary output of the class. |

## Integration Patterns

### Standard Usage
The TagSetLookupTable is intended to be used once during a loading sequence. A manager class will prepare the necessary data, create an instance to perform the resolution, and cache the result for engine-wide use.

```java
// Within a TagSetManager or similar system during initialization
Map<String, MyTagSet> rawTagSets = loadAllTagSetsFromAssets();
Object2IntMap<String> tagSetIndices = buildIndexFor(rawTagSets.keySet());
Object2IntMap<String> tagIndices = buildIndexForAllKnownTags();

// The table is created to perform the one-time computation
TagSetLookupTable<MyTagSet> resolver = new TagSetLookupTable<>(
    rawTagSets,
    tagSetIndices,
    tagIndices
);

// The result is cached for fast runtime access
this.runtimeTagLookup = resolver.getFlattenedSet();
```

### Anti-Patterns (Do NOT do this)
-   **Per-Query Instantiation:** Never create a new TagSetLookupTable for each tag query. The constructor performs a very expensive graph traversal and is designed to be called only once per data set.
-   **External Mutation:** Do not modify the map or sets returned by `getFlattenedSet`. The integrity of the engine's tag system relies on this data being treated as immutable. Modifying it will lead to inconsistent and undefined behavior across the application.
-   **Concurrent Construction:** Do not share the input maps with other threads while a TagSetLookupTable is being constructed. The process is not thread-safe and will fail unpredictably.

## Data Pipeline
The class functions as a processing stage in the engine's asset loading pipeline, transforming raw configuration data into an optimized in-memory representation.

> Flow:
> Raw TagSet Files (JSON) -> Asset Parser -> `Map<String, T>` & `Object2IntMap` -> **TagSetLookupTable** -> `Int2ObjectMap<IntSet>` (Cached) -> Runtime Systems (e.g., Physics, Crafting)

