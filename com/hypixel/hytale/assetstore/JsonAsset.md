---
description: Architectural reference for the JsonAsset interface, the core contract for all assets loaded from JSON.
---

# JsonAsset

**Package:** com.hypixel.hytale.assetstore
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface JsonAsset<K> {
```

## Architecture & Concepts
The JsonAsset interface is a foundational contract within the Hytale Asset Pipeline. It establishes a standardized mechanism for identifying and retrieving assets that are defined and loaded from JSON files. Any class representing a game asset deserialized from JSON, such as a block definition, item metadata, or sound configuration, **must** implement this interface.

Its primary architectural role is to decouple the AssetManager from the concrete types of assets it manages. By enforcing the presence of a unique identifier via the getId method, the AssetManager can build a generic, type-safe caching and retrieval system. The generic type parameter K allows for flexibility in the keying strategy, supporting identifiers of type String, Integer, or a more complex composite key object.

This interface is the bridge between raw data on disk and a managed, identifiable object in memory.

### Lifecycle & Ownership
As an interface, JsonAsset has no lifecycle of its own. The following rules apply to all implementing classes.

- **Creation:** Instances are almost exclusively created by the asset deserialization pipeline, typically involving a library like Gson. The engine reads a JSON file from a content pack, and the deserializer constructs the object in memory. Manual instantiation is strongly discouraged.
- **Scope:** The lifetime of a JsonAsset object is tied to the AssetManager's cache. It is loaded on demand (or during a pre-loading phase) and persists in memory as long as it is referenced by the cache or other game systems.
- **Destruction:** Objects are eligible for garbage collection when they are evicted or cleared from the AssetManager's cache and no other strong references exist. This typically occurs during a world unload or a resource pack change.

## Internal State & Concurrency
The interface itself is stateless. However, implementers must adhere to strict concurrency and state management rules.

- **State:** The value returned by getId **must** be immutable or derived from immutable fields. It is used as a key in concurrent collections and its value must not change after the object has been constructed. The object's state is generally considered "write-once" upon deserialization.
- **Thread Safety:** Implementations of getId **must** be thread-safe and non-blocking. It is expected to be a simple field accessor. The method will be called from various engine threads, including the main game loop, rendering threads, and background loading threads.

## API Surface
The public contract is minimal but critical for the asset system's integrity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | K | O(1) | Returns the unique, immutable identifier for this asset. This value is used as the primary key in the AssetManager cache. |

## Integration Patterns

### Standard Usage
Developers should never implement or instantiate JsonAsset derivatives directly. Instead, they are retrieved from the central AssetManager, which handles the entire loading and deserialization lifecycle.

```java
// Correctly retrieve a managed asset that implements JsonAsset
AssetManager assetManager = context.getService(AssetManager.class);
BlockDefinition stone = assetManager.get("hytale:stone", BlockDefinition.class);

// The engine can now use the contract to get its ID
String id = stone.getId(); // Returns "hytale:stone"
```

### Anti-Patterns (Do NOT do this)
- **Stateful ID:** Do not implement getId to compute a value from mutable state. This will corrupt the AssetManager's cache and lead to unpredictable behavior.
- **Manual Instantiation:** Avoid creating instances with `new`. These objects will not be tracked by the engine's asset management system, will not be hot-reloadable, and will not be available to other systems that rely on the central asset registry.

## Data Pipeline
The JsonAsset interface is a key component in the transformation of on-disk data into a usable in-memory object.

> Flow:
> JSON File on Disk -> Asset Deserializer (Gson) -> **JsonAsset Instance** -> AssetManager Cache -> Game Systems

