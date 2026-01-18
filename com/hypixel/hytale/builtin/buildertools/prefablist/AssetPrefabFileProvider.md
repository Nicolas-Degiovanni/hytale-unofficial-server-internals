---
description: Architectural reference for AssetPrefabFileProvider
---

# AssetPrefabFileProvider

**Package:** com.hypixel.hytale.builtin.buildertools.prefablist
**Type:** Transient

## Definition
```java
// Signature
public class AssetPrefabFileProvider implements FileListProvider {
```

## Architecture & Concepts

The AssetPrefabFileProvider is a concrete implementation of the FileListProvider interface. Its primary architectural role is to serve as a data source for user-facing UI components, specifically file browsers integrated within the Hytale builder tools. It is responsible for discovering, listing, and searching for prefab asset files (`.prefab.json`).

This class creates a virtual filesystem abstraction over the physical asset directories scattered across different content packs. The root of this virtual filesystem is a list of all detected asset packs. Navigating into a pack reveals its internal directory structure and prefab files. This abstraction decouples the UI from the underlying on-disk storage layout of game assets.

The provider's source of truth is the **PrefabStore** singleton. It queries the PrefabStore to locate the root paths of all available asset packs, ensuring it always operates on the currently loaded set of game data. All file operations, including directory traversal and recursive searching, are handled directly by this class using the Java NIO file APIs.

## Lifecycle & Ownership

-   **Creation:** An instance of AssetPrefabFileProvider is typically created on-demand by a higher-level UI controller or service when a prefab browser UI is initialized. It is not a singleton and should be treated as a short-lived object.
-   **Scope:** The object's lifetime is scoped to the UI component that consumes it. It persists only as long as the file browser is open and active.
-   **Destruction:** The object holds no persistent resources or file handles. It is eligible for garbage collection as soon as the UI component is destroyed and all references to the instance are released. No explicit cleanup is required.

## Internal State & Concurrency

-   **State:** This class is entirely **stateless**. It maintains no internal caches or state between method invocations. Every call to its public methods, particularly getFiles, results in direct and fresh I/O operations against the filesystem. This design guarantees that the UI always displays the most current data but can introduce performance latency.

-   **Thread Safety:** This implementation is **not thread-safe** and is fundamentally blocking. Its methods perform synchronous disk I/O and are designed to be invoked from a dedicated UI thread or a background worker thread.

    **Warning:** Calling getFiles from multiple threads concurrently is unsupported. While the stateless nature prevents data corruption, it can lead to severe performance degradation due to disk thrashing. All interactions with an instance of this class should be serialized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFiles(currentDir, searchQuery) | List | O(N * M) | The primary entry point. Fetches file listings or search results. Complexity is high due to recursive file traversal (N files) and fuzzy string matching (M query length). **This is a blocking I/O operation.** |
| resolveVirtualPath(virtualPath) | Path | O(P) | Translates a virtual path (e.g., pack/folder/file) into an absolute filesystem Path. Complexity depends on the number of asset packs (P) to search through. Returns null if the path is invalid. |
| getPackDisplayName(packKey) | String | O(P) | Retrieves the user-friendly display name for a given asset pack key. |

## Integration Patterns

### Standard Usage

The AssetPrefabFileProvider is designed to be injected into a generic UI component that operates on the FileListProvider contract. The UI controller instantiates the provider and uses it to populate the view in response to user interactions.

```java
// In a UI Controller or Service
FileListProvider provider = new AssetPrefabFileProvider();

// User navigates to the root directory
List<FileListProvider.FileEntry> rootPacks = provider.getFiles(Path.of(""), "");

// User searches for "castle"
List<FileListProvider.FileEntry> searchResults = provider.getFiles(Path.of(""), "castle");
```

### Anti-Patterns (Do NOT do this)

-   **Long-Lived Caching:** Do not cache the `List<FileEntry>` results returned by this provider in a long-lived service. The provider is designed to reflect the current state of the filesystem. If caching is required, implement a separate CachingFileListProvider decorator that wraps this class.
-   **Invocation from Game Loop:** Never call getFiles or other methods from a performance-critical code path like the main game update loop. The associated blocking I/O will cause significant frame rate drops and application stutter. Defer all calls to a background thread or an asynchronous task.
-   **External Instantiation:** While technically possible, avoid creating instances of this class outside of the UI layer it is designed to serve. It is not a general-purpose file utility.

## Data Pipeline

As a data source, this class typically initiates a data flow rather than processing one. The flow begins with a user action and ends with a UI update.

> Flow:
> User Input (e.g., typing in search bar) -> UI Event -> **AssetPrefabFileProvider.getFiles()** -> Filesystem Read -> `List<FileEntry>` -> UI Component Rendering

