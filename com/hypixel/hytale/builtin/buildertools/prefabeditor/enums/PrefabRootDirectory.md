---
description: Architectural reference for PrefabRootDirectory
---

# PrefabRootDirectory

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.enums
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum PrefabRootDirectory {
```

## Architecture & Concepts
PrefabRootDirectory is a type-safe enumeration that serves as a high-level abstraction over the physical storage locations of game prefabs. It is the authoritative source for resolving the root paths for different categories of prefabs, such as those specific to a server, bundled in asset packs, or used for world generation.

The core architectural choice is the use of a `Supplier<Path>` for path resolution instead of a direct `Path` field. This decouples the enum's static definition from the runtime state of the engine. Path resolution is deferred until a path is explicitly requested, allowing the underlying `PrefabStore` service to initialize fully before its paths are queried.

This enum acts as a configuration model, primarily for the in-game Prefab Editor. It provides not only the file paths but also associated metadata required for UI rendering, such as localization strings and capability flags like `supportsMultiPack`. This centralizes the logic for locating prefabs and prevents path resolution logic from being scattered across the UI and tool-related codebase.

## Lifecycle & Ownership
- **Creation:** Instances of this enum (SERVER, ASSET, etc.) are constructed and initialized by the Java Virtual Machine during class loading. This occurs once, very early in the server's startup sequence.
- **Scope:** As a static enum, all instances persist for the entire lifetime of the application. They are effectively global, immutable singletons.
- **Destruction:** Instances are reclaimed by the JVM only upon application shutdown. There is no mechanism for manual destruction or reloading.

## Internal State & Concurrency
- **State:** The enum instances themselves are **immutable**. All fields are final and assigned during static initialization. However, this class is a facade over the `PrefabStore` singleton. The results of methods like `getPrefabPath` are dependent on the state of `PrefabStore`, which is considered the single source of truth.

- **Thread Safety:** This enum is conditionally thread-safe. While its own state is immutable, its methods delegate calls to `PrefabStore.get()`. Therefore, safe concurrent access to `PrefabRootDirectory` is entirely contingent on the thread safety of the `PrefabStore` implementation.

    **Warning:** Callers must ensure that any concurrent access to this enum's methods is synchronized if the underlying `PrefabStore` is not guaranteed to be thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPrefabPath() | Path | O(1) | Resolves and returns the root path for this directory type by invoking the underlying path supplier. Throws if `PrefabStore` is not initialized. |
| getLocalizationString() | String | O(1) | Returns the static localization key used for displaying the directory name in user interfaces. |
| supportsMultiPack() | boolean | O(1) | Returns true if this directory type represents a collection of multiple asset packs, false otherwise. |
| getAllPrefabPaths() | List | O(N) | Returns a complete list of all specific prefab paths for this root. If `supportsMultiPack` is true, N is the number of loaded asset packs. Otherwise, N is 1. |

## Integration Patterns

### Standard Usage
This enum is intended to be used by systems, like the Prefab Editor, that need to enumerate all possible prefab locations for a given category. The primary entry point for this is the `getAllPrefabPaths` method.

```java
// Used by the Prefab Editor UI to populate its file browser.
// This retrieves all paths from all loaded asset packs.
List<PrefabStore.AssetPackPrefabPath> assetPaths = PrefabRootDirectory.ASSET.getAllPrefabPaths();

for (PrefabStore.AssetPackPrefabPath path : assetPaths) {
    // UI can now display the pack name and scan the associated path
    // for .pfb files.
    System.out.println("Scanning " + path.getPath() + " from pack: " + path.getPackName());
}
```

### Anti-Patterns (Do NOT do this)
- **Path Caching:** Do not call `getPrefabPath` once and store the result for the entire session. The `PrefabStore` is the authoritative source. While its state is unlikely to change post-initialization, always query the enum to respect the intended design and prevent stale data.
- **Logic Replication:** Do not attempt to manually construct paths to prefab directories. This enum is the single source of truth for locating prefab roots. Bypassing it will lead to brittle code that breaks when asset loading logic changes.

## Data Pipeline
PrefabRootDirectory acts as a configuration source rather than a data processor. It initiates data discovery by providing paths to other systems.

> Flow:
> UI System Request -> **PrefabRootDirectory.getAllPrefabPaths()** -> PrefabStore Singleton -> Filesystem Scanner -> List of Prefab Files

