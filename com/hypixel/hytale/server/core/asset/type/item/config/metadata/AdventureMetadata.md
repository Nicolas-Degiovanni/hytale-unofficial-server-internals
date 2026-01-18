---
description: Architectural reference for AdventureMetadata
---

# AdventureMetadata

**Package:** com.hypixel.hytale.server.core.asset.type.item.config.metadata
**Type:** Data Model / Configuration Fragment

## Definition
```java
// Signature
public class AdventureMetadata {
```

## Architecture & Concepts
AdventureMetadata is a data model class that represents a specific, typed fragment of an item's configuration. It is not a standalone service but rather a structured data container used by the server's asset system. Its primary role is to hold data deserialized from on-disk asset files, such as a JSON definition for a sword or potion.

This class is a component of the Hytale Codec framework, a powerful serialization system responsible for loading game data. The static **CODEC** and **KEYED_CODEC** fields are the most critical aspects of this class; they define the schema for how "Adventure" metadata is read from and written to asset files.

The codec's use of **appendInherited** signifies its participation in a configuration inheritance system. This allows game designers to create base item templates and have more specific items inherit and selectively override their properties. AdventureMetadata, therefore, acts as a leaf node in a larger configuration tree, representing a single, serializable trait an item can possess.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Codec framework during the asset loading phase. A central `AssetManager` or `ConfigurationLoader` service reads an item definition file and uses the static **KEYED_CODEC** to find the relevant data block and construct an AdventureMetadata object. Direct instantiation by developers is an anti-pattern.
- **Scope:** The lifetime of an AdventureMetadata instance is strictly bound to its parent `ItemConfiguration` object. It is loaded into memory when the server initializes or loads a world and persists as long as the parent item asset is required.
- **Destruction:** The object is marked for garbage collection when its parent `ItemConfiguration` is unloaded, typically upon server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** Mutable. The class contains a single boolean field, *cursed*, which directly reflects the configured value from an asset file. It is a simple data-holder and performs no caching or complex state management.
- **Thread Safety:** **Not thread-safe.** This object is not designed for concurrent access. It is populated on a single thread during the asset loading pipeline. Subsequent reads from game logic threads are generally safe, but any modification after initialization is a severe anti-pattern and will introduce race conditions. Configuration data should be treated as immutable after the initial load.

## API Surface
The primary public contract for this class is its static Codec, used by the asset system. The instance methods are simple accessors for the deserialized data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isCursed() | boolean | O(1) | Returns true if the item is configured with the "cursed" property. |
| setCursed(boolean) | void | O(1) | **WARNING:** For internal framework use only. Modifies the cursed state. |

## Integration Patterns

### Standard Usage
Developers should never create or manage this class directly. Instead, it should be retrieved from a higher-level asset definition object, which encapsulates all its metadata components.

```java
// Correctly retrieve the metadata from a parent ItemConfiguration
ItemConfiguration itemConfig = itemRegistry.get("my_cursed_sword");
AdventureMetadata adventureMeta = itemConfig.getMetadata(AdventureMetadata.KEYED_CODEC);

if (adventureMeta != null && adventureMeta.isCursed()) {
    // Apply cursed effect logic
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new AdventureMetadata()`. This creates an uninitialized object that is disconnected from the asset management system and will not reflect the actual game configuration.
- **Post-Load Modification:** Do not call `setCursed()` on a loaded metadata instance. This object represents static configuration data, not dynamic runtime state. Modifying it after the server has started can lead to inconsistent behavior and non-deterministic bugs.

## Data Pipeline
AdventureMetadata is the terminal endpoint in a data flow that begins with a raw asset file on disk. The Hytale Codec framework orchestrates the transformation of this raw data into a strongly-typed, in-memory Java object.

> Flow:
> Item Asset File (JSON) -> Asset Loading Service -> Hytale Codec Deserializer -> **AdventureMetadata instance** -> Attached to ItemConfiguration -> Queried by Game Logic Systems

