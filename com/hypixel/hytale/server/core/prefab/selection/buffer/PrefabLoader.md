---
description: Architectural reference for PrefabLoader
---

# PrefabLoader

**Package:** com.hypixel.hytale.server.core.prefab.selection.buffer
**Type:** Utility

## Definition
```java
// Signature
public class PrefabLoader {
```

## Architecture & Concepts
The PrefabLoader is a critical file system abstraction utility responsible for translating logical, dot-separated prefab names into concrete file system paths. It serves as the primary bridge between the server's conceptual understanding of a prefab (e.g., *hytale.world.zone1.special_chest*) and its physical location on disk (e.g., *.../hytale/world/zone1/special_chest.prefab.json*).

This class decouples higher-level systems, such as the PrefabBuffer, from the underlying operating system's file path conventions. It provides two core resolution strategies:
1.  **Direct Resolution:** Locating a single, specific prefab file.
2.  **Wildcard Resolution:** Discovering all prefab files within a given directory namespace using a `.*` suffix.

Furthermore, it provides the inverse utility, **resolveRelativeJsonPath**, to convert a file system path back into its canonical, dot-separated key. This is essential for registering discovered prefabs into caches and registries under their correct logical names.

## Lifecycle & Ownership
- **Creation:** PrefabLoader instances are typically created on-demand by services that need to perform multiple prefab lookups within a consistent root directory. The static methods can be invoked at any time without instantiation, making the class highly flexible. It is a transient, lightweight object.
- **Scope:** The lifetime of an instance is bound to its creator. As it holds no managed resources and its state is immutable, it is cheap to create and discard. It is common for an instance to exist only for the duration of a single bulk-loading operation.
- **Destruction:** The object is managed by the Java garbage collector and is reclaimed when all references are lost. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** An instance of PrefabLoader is effectively immutable. Its only state, the **rootFolder** path, is final and set during construction. The class performs no internal caching; every method call results in a direct file system query.
- **Thread Safety:** This class is thread-safe. The instance state is immutable, and its static methods operate exclusively on their arguments. File system operations are delegated to the underlying Java NIO APIs, which are themselves thread-safe. Multiple threads can safely and concurrently invoke resolution methods on the same or different instances.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| resolvePrefabs(String, Consumer) | void | O(N) | Resolves a prefab name to one or more file paths. Throws IOException on failure. Complexity is O(1) for direct lookups and O(N) for wildcard searches, where N is the number of files in the subtree. |
| resolveRelativeJsonPath(String, Path, Path) | String | O(L) | Converts a file system path back to a canonical dot-separated prefab name. L is the length of the path string. Throws IllegalArgumentException if the path is inconsistent with the name. |

## Integration Patterns

### Standard Usage
The primary use case is to discover all prefabs within a given namespace (e.g., a mod's content folder) and pass them to a consumer for processing, such as loading them into a cache.

```java
// A service needs to load all prefabs from the "hytale.world.dungeons" namespace
Path serverRoot = Paths.get("./server/assets/prefabs");
PrefabLoader loader = new PrefabLoader(serverRoot);

List<Path> foundPrefabs = new ArrayList<>();
try {
    // Use a wildcard to find all prefabs in the dungeons directory and its subdirectories
    loader.resolvePrefabs("hytale.world.dungeons.*", foundPrefabs::add);
} catch (IOException e) {
    // Handle critical file system errors, e.g., directory not found
    log.error("Failed to resolve dungeon prefabs", e);
}

// The foundPrefabs list now contains Path objects for all matching prefabs
```

### Anti-Patterns (Do NOT do this)
- **Manual Path Concatenation:** Do not manually build file paths by replacing dots with slashes. This bypasses the class's validation and error handling, making the system brittle. Always use **resolvePrefabs**.
- **Ignoring IOExceptions:** File system operations are inherently unreliable. Calls to **resolvePrefabs** are decorated with **throws IOException** for a reason. Failure to catch **NoSuchFileException** or **NotDirectoryException** can lead to unhandled server exceptions.
- **Incorrect Wildcard Usage:** The wildcard `.*` is only valid as a suffix. Using it in the middle of a prefab name (e.g., `hytale.*.dungeons`) is not supported and will result in a **NoSuchFileException**.

## Data Pipeline
The PrefabLoader is a data transformation component that converts logical identifiers into physical resource locators.

> Flow:
> Logical Prefab Name (String) -> **PrefabLoader.resolvePrefabs** -> File System Query -> Concrete File Paths (Consumer<Path>)

