---
description: Architectural reference for PrefabStoreRoot
---

# PrefabStoreRoot

**Package:** com.hypixel.hytale.server.worldgen.prefab
**Type:** Type-Safe Enum / Utility

## Definition
```java
// Signature
public enum PrefabStoreRoot {
```

## Architecture & Concepts
The PrefabStoreRoot enum serves as a high-level abstraction to disambiguate between the different physical locations where game prefabs can be stored. Its primary architectural function is to act as a centralized and type-safe strategy selector for path resolution, preventing the spread of hardcoded file paths and conditional logic throughout the server's world generation and asset systems.

It defines two fundamental sources for prefabs:
*   **ASSETS:** Represents prefabs that are an integral part of the core game distribution. These are typically read-only, managed by the global PrefabStore, and are expected to be universally available.
*   **WORLD_GEN:** Represents prefabs that are specific to a particular world instance. These are located within the world's data folder, allowing for server-specific customization, additions, or overrides.

By encapsulating this distinction, the enum provides a single, authoritative point of control for determining the correct prefab directory, simplifying any system that needs to load or interact with prefabs.

## Lifecycle & Ownership
As a Java enum, PrefabStoreRoot has a lifecycle strictly managed by the JVM, ensuring its stability and global availability.

-   **Creation:** The enum instances, ASSETS and WORLD_GEN, are constructed automatically by the JVM when the PrefabStoreRoot class is first loaded and initialized. This process is guaranteed to happen only once.
-   **Scope:** The instances are static, final, and globally accessible. They persist for the entire lifetime of the server application.
-   **Destruction:** The instances are garbage collected only when the application's class loader is unloaded, which effectively occurs during server shutdown.

## Internal State & Concurrency
-   **State:** **Immutable**. Enum constants are inherently final. The class holds no mutable instance or static state. Its behavior is determined entirely by its inputs and the state of external systems like the global PrefabStore.
-   **Thread Safety:** **Fully thread-safe**. Due to its immutable nature and the use of a static, pure function for its core logic, this enum can be safely accessed and used by any number of threads concurrently without requiring any external synchronization.

## API Surface
The public contract is primarily the static resolution method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| resolvePrefabStore(store, dataFolder) | Path | O(1) | Resolves the absolute filesystem path for a given prefab store root. This is the primary entry point for this utility. |

## Integration Patterns

### Standard Usage
The intended use is to call the static resolvePrefabStore method to obtain a Path object, which can then be passed to a file loader or asset manager.

```java
// In a world generation service, obtain the server's data folder
Path serverDataFolder = serverContext.getDataFolderPath();

// Resolve the path for world-specific prefabs using the WORLD_GEN constant
Path worldPrefabsPath = PrefabStoreRoot.resolvePrefabStore(PrefabStoreRoot.WORLD_GEN, serverDataFolder);

// Resolve the path for core game asset prefabs
// The dataFolder parameter is ignored in this case but should still be provided
Path assetPrefabsPath = PrefabStoreRoot.resolvePrefabStore(PrefabStoreRoot.ASSETS, serverDataFolder);
```

### Anti-Patterns (Do NOT do this)
-   **Manual Path Reconstruction:** Do not replicate the internal switch logic. The enum is the single source of truth for path resolution. Manually building the path creates brittle code that will break if the internal storage strategy changes.

    ```java
    // BAD: Duplicates internal logic and is prone to error
    Path myPath;
    if (root == PrefabStoreRoot.WORLD_GEN) {
        myPath = dataFolder.resolve("Prefabs"); // Magic string
    }
    ```

-   **Null dataFolder Argument:** While the ASSETS case may not use the dataFolder path, the WORLD_GEN case critically depends on it. Passing a null or invalid path will result in a NullPointerException or incorrect behavior. Always provide a valid data folder path.

## Data Pipeline
PrefabStoreRoot acts as a configuration resolver at the beginning of any prefab loading pipeline. It does not process data itself but rather provides the initial location from which to source it.

> Flow:
> World Generation Service -> **PrefabStoreRoot.resolvePrefabStore** -> Filesystem Path -> Prefab Loader -> Prefab Data

