---
description: Architectural reference for TagPattern
---

# TagPattern

**Package:** com.hypixel.hytale.server.core.asset.type.tagpattern.config
**Type:** Asset Type

## Definition
```java
// Signature
public abstract class TagPattern
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, TagPattern>>,
   NetworkSerializable<com.hypixel.hytale.protocol.TagPattern> {
```

## Architecture & Concepts
The TagPattern class is an abstract base for a polymorphic asset-driven rule system. It defines the fundamental contract for matching a complex set of tags against a predefined pattern. These patterns are defined as data in JSON files and are loaded, managed, and indexed by the Hytale AssetStore framework.

Architecturally, TagPattern serves as a data-driven predicate. Game systems, such as mob spawners, loot generators, or quest progression logic, can retrieve a specific TagPattern instance from the central AssetStore and use its *test* method to evaluate complex conditions without hard-coding the logic.

Its implementation of JsonAssetWithMap integrates it directly into the asset pipeline, allowing designers and developers to define new patterns as simple data files. The NetworkSerializable interface ensures that these server-defined rules can be efficiently synchronized with the client, likely for predictive or UI-related purposes.

### Lifecycle & Ownership
- **Creation:** Instances of TagPattern subclasses are not created directly via a constructor. They are instantiated by the AssetStore's deserialization pipeline, specifically through the static CODEC field, when the server boots and loads all game assets from disk.
- **Scope:** A TagPattern instance is a global, application-scoped object. Once loaded by the AssetStore, it persists for the entire server session.
- **Destruction:** Instances are destroyed only when the AssetStore is cleared, which typically occurs during a full server shutdown. The Java garbage collector reclaims the memory.

## Internal State & Concurrency
- **State:** The primary state, including its unique *id*, is immutable after being loaded from its source JSON file. However, the class contains a mutable cache field, *cachedPacket*, which is a SoftReference to a network-optimized representation of the pattern. This is a performance optimization to avoid repeated serialization.

- **Thread Safety:** This class is **not inherently thread-safe**.
    - The static `getAssetStore` method uses a non-atomic check-then-act pattern for lazy initialization. Calling this method from multiple threads concurrently before the ASSET_STORE is initialized will result in a race condition. It is designed to be called from a single main thread during the server's bootstrap sequence.
    - The `cachedPacket` field is mutable. If multiple threads attempt to generate the network packet for the same TagPattern instance simultaneously, it could lead to race conditions. Access to this field or its generating logic must be externally synchronized.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the singleton AssetStore responsible for all TagPattern assets. **Warning:** Not thread-safe on first call. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Provides direct access to the indexed map of all loaded TagPattern assets. |
| getId() | String | O(1) | Returns the unique asset identifier for this pattern, derived from its file name. |
| test(Int2ObjectMap) | abstract boolean | O(N) | The core evaluation method. Returns true if the provided tags match the pattern's logic. Complexity is dependent on the concrete implementation. |

## Integration Patterns

### Standard Usage
The intended use is to retrieve a pre-loaded pattern from the central AssetStore using its unique string identifier and then invoke the test method.

```java
// How a developer should normally use this
// Assume 'entityTags' is an Int2ObjectMap<IntSet> representing an entity's tags.
IndexedLookupTableAssetMap<String, TagPattern> patterns = TagPattern.getAssetMap();
TagPattern rareMobPattern = patterns.get("hytale:rare_golem_spawn_condition");

if (rareMobPattern != null && rareMobPattern.test(entityTags)) {
    // Spawn the rare golem
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** It is impossible to instantiate TagPattern as it is abstract. Concrete subclasses must **never** be instantiated with *new*. The AssetStore is the sole owner and creator of these objects.
- **Concurrent Initialization:** Do not call `TagPattern.getAssetStore()` from multiple threads during server startup. This will lead to unpredictable behavior and potential data corruption. Ensure it is called from a single, controlled initialization thread first.
- **State Mutation:** Do not attempt to modify the state of a TagPattern instance after it has been loaded. These assets are treated as immutable singletons by the rest of the engine.

## Data Pipeline
The TagPattern class is involved in two primary data flows: asset loading and runtime evaluation.

> **Asset Loading Flow:**
> JSON File on Disk -> AssetStore Loader -> `CODEC` Deserializer -> **TagPattern Instance** -> IndexedLookupTableAssetMap Registry

> **Runtime Evaluation Flow:**
> Game System Logic -> `TagPattern.getAssetMap().get(id)` -> **TagPattern.test(tags)** -> Boolean Result -> Game State Change

