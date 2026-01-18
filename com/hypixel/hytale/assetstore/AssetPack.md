---
description: Architectural reference for AssetPack
---

# AssetPack

**Package:** com.hypixel.hytale.assetstore
**Type:** Transient Data Object

## Definition
```java
// Signature
public class AssetPack {
```

## Architecture & Concepts
The AssetPack class is a fundamental data structure within the Hytale Asset Subsystem. It serves as an in-memory representation of a physical collection of game assets, such as a directory on disk or a compressed archive file (e.g., a ZIP file). Its primary architectural role is to provide a unified abstraction over different storage formats.

This abstraction is critical for the engine's flexibility, allowing the AssetStore to treat a loose collection of files in a development environment and a packed, distributable archive in a production build identically. Each AssetPack encapsulates not only the path to the assets but also associated metadata, such as its name, its manifest, and a reference to a virtual Java NIO FileSystem if it represents an archive.

This class is not a service or a manager; it is a passive, immutable data container created and managed by the AssetStore during the engine's asset discovery phase.

### Lifecycle & Ownership
- **Creation:** AssetPack instances are exclusively instantiated by a factory or loader class, such as an AssetPackLoader, during the game's startup or when resource packs are reloaded. The loader scans designated directories, identifies valid packs, parses their manifests, and creates corresponding AssetPack objects.
- **Scope:** An AssetPack object persists as long as it is registered within the central AssetStore. Typically, this means its lifetime is tied to the application session.
- **Destruction:** The object is eligible for garbage collection when the AssetStore is cleared or re-initialized, which occurs during client shutdown or a full resource reload event. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The AssetPack is **strictly immutable**. All its internal fields are final and are assigned only once during construction. The state represents a snapshot of the asset source at the time of its loading. The boolean field isImmutable refers to the underlying physical asset source, not the object itself.
- **Thread Safety:** This class is **inherently thread-safe**. Due to its immutable design, multiple threads (e.g., the main game thread and background asset loading threads) can safely access an AssetPack instance without any need for external synchronization or locks. This is a deliberate design choice to support high-performance, concurrent asset streaming.

## API Surface
The public contract consists entirely of non-blocking data accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getName() | String | O(1) | Returns the unique identifier for the pack. |
| getRoot() | Path | O(1) | Returns the root path for assets within this pack. |
| getFileSystem() | FileSystem | O(1) | Returns the virtual NIO FileSystem if the pack is an archive; null otherwise. |
| getManifest() | PluginManifest | O(1) | Returns the parsed manifest metadata associated with this pack. |
| isImmutable() | boolean | O(1) | Indicates if the underlying asset source on disk is considered read-only. |
| getPackLocation() | Path | O(1) | Returns the absolute path to the pack itself (the folder or archive file). |

## Integration Patterns

### Standard Usage
The AssetPack is not typically used directly. Instead, systems interact with the AssetStore, which uses its collection of AssetPack objects internally to resolve asset requests. A system requiring direct access would retrieve the list of active packs from a higher-level service.

```java
// Example of an asset resolver iterating through packs
AssetStore store = context.getService(AssetStore.class);
List<AssetPack> allPacks = store.getRegisteredPacks();

for (AssetPack pack : allPacks) {
    // Use pack.getRoot() to build a full path to a potential asset
    Path potentialAsset = pack.getRoot().resolve("textures/blocks/stone.png");
    if (Files.exists(potentialAsset)) {
        // Asset found, proceed to load
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new AssetPack(...)`. Doing so will create an object unknown to the engine's AssetStore, rendering it invisible to the asset resolution pipeline. This will lead to assets failing to load and unpredictable behavior.
- **State Assumption:** Do not assume `getFileSystem()` will be non-null. Code must be robust enough to handle both archive-based packs (which have a FileSystem) and directory-based packs (which do not). Failure to check for null will result in NullPointerExceptions.

## Data Pipeline
The AssetPack acts as a data source provider within the larger asset resolution pipeline. It does not process data itself but provides the necessary context for other systems to locate and read asset data.

> Flow:
> Asset Request (e.g., "hytale:stone_block") -> AssetStore -> **AssetPack** (Provides root path and FileSystem) -> PathResolver -> File I/O Stream -> Asset Deserializer -> Engine Object

