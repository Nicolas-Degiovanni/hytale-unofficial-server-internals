---
description: Architectural reference for AssetModule
---

# AssetModule

**Package:** com.hypixel.hytale.server.core.asset
**Type:** Singleton

## Definition
```java
// Signature
public class AssetModule extends JavaPlugin {
```

## Architecture & Concepts
The AssetModule is a core server plugin that serves as the central nervous system for asset management. It is responsible for the discovery, registration, and lifecycle management of all game assets, which are bundled into logical units called AssetPacks.

Its primary role is to act as the bridge between the server's file system and the in-memory asset database, the AssetRegistry. During server initialization, the AssetModule scans predefined directories (e.g., mods, assets) for valid AssetPacks, which can be either directories or zip archives. For each discovered pack, it parses its manifest, validates its configuration, and registers it with the system.

The module orchestrates the main asset loading sequence, which is triggered by the dispatch of a single, critical LoadAssetEvent. Upon receiving this event, it iterates through all registered AssetPacks and delegates the low-level file reading and data parsing to the AssetRegistryLoader. This ensures that all assets are loaded in a deterministic order and that the global asset state is protected from race conditions via a global write lock.

Furthermore, the AssetModule provides an optional file monitoring service, the AssetMonitor, which enables hot-reloading of assets during development. This allows for real-time changes to asset files without requiring a full server restart, dramatically improving iteration speed.

## Lifecycle & Ownership
- **Creation:** The AssetModule is instantiated once by the PluginManager during the server's bootstrap sequence. Its constructor is called with a JavaPluginInit context, and it immediately registers itself as the global singleton instance. Direct instantiation outside of the plugin system is strictly forbidden.
- **Scope:** The AssetModule instance is a session-scoped singleton. It lives for the entire duration of the server process, from startup to shutdown.
- **Destruction:** The shutdown method is invoked by the PluginManager during the server shutdown process. This method is responsible for releasing all managed resources, which critically includes shutting down the AssetMonitor thread and closing any open FileSystem handles associated with zipped AssetPacks.

## Internal State & Concurrency
- **State:** The AssetModule maintains a mutable state representing the currently loaded asset sources.
    - **assetPacks:** A list of all registered AssetPack objects. This collection is the single source of truth for which asset sources are active.
    - **pendingAssetStores:** A temporary buffer for AssetStores that are registered after the initial bulk-load event. This ensures they are correctly initialized.
    - **hasLoaded:** A critical boolean flag that functions as a latch, preventing the initial, destructive asset loading process from running more than once.
- **Thread Safety:** This class is designed to be thread-safe within the engine's concurrency model.
    - It uses thread-safe collections like CopyOnWriteArrayList for its internal lists, allowing for safe concurrent reads.
    - **CRITICAL:** All operations that mutate the global asset state (e.g., loading assets, registering a new pack post-boot) are performed while holding the global `AssetRegistry.ASSET_LOCK` write lock. This serializes all modifications to the asset database, preventing data corruption and race conditions. Public methods that trigger these operations, such as registerPack and unregisterPack, handle locking internally.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | AssetModule | O(1) | Statically retrieves the global singleton instance. |
| getBaseAssetPack() | AssetPack | O(1) | Returns the primary asset pack, typically containing the base game assets. |
| getAssetPacks() | List<AssetPack> | O(1) | Returns a thread-safe list of all currently registered asset packs. |
| registerPack(name, path, manifest) | void | O(N) | Registers a new asset pack. Involves I/O and acquires a global write lock. Dispatches an AssetPackRegisterEvent if assets have already been loaded. |
| unregisterPack(name) | void | O(N) | Unregisters an existing asset pack by name, closing its resources. Acquires a global write lock and dispatches an AssetPackUnregisterEvent. |
| findAssetPackForPath(path) | AssetPack | O(N) | Locates the asset pack that contains the given absolute file path. Returns null if not found. |
| isAssetPathImmutable(path) | boolean | O(N) | Checks if a file path belongs to an immutable (e.g., zipped) asset pack. |

## Integration Patterns

### Standard Usage
Developers typically do not interact with the AssetModule directly. The primary integration is declarative: by placing a properly formatted mod or asset pack into the server's `mods` directory, the AssetModule will automatically discover and load it on startup.

For advanced plugins that need to manage asset packs dynamically, the global instance should be retrieved to register or unregister packs programmatically.

```java
// Example of a plugin dynamically loading a new asset pack
AssetModule assetModule = AssetModule.get();
Path newPackPath = Paths.get("dynamic_content.zip");
PluginManifest manifest = loadMyManifest(newPackPath); // User-defined method

// This will trigger the loading of all assets within the new pack
assetModule.registerPack("my_dynamic_pack", newPackPath, manifest);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new AssetModule()`. The plugin system manages its lifecycle. Use `AssetModule.get()` to retrieve the singleton instance.
- **Manual Lifecycle Management:** Do not call the `setup()` or `shutdown()` methods. These are invoked exclusively by the PluginManager.
- **Unsynchronized State Modification:** Do not attempt to modify the list returned by `getAssetPacks()` directly. Use the `registerPack` and `unregisterPack` methods to ensure that proper locking and event dispatching occur. Failure to do so will lead to severe concurrency issues and data corruption.
- **Premature Asset Access:** Do not attempt to access assets from an AssetStore before the `LoadAssetEvent` has been fully processed. The `hasLoaded` flag is an internal guard for this, but external code should rely on engine-level lifecycle events to determine when assets are ready.

## Data Pipeline

> Flow (Initial Server Boot):
> Server Bootstrap -> PluginManager loads **AssetModule** -> **AssetModule.setup()** -> Scans file system for packs -> **AssetModule.registerPack()** for each -> Engine dispatches LoadAssetEvent -> **AssetModule** handler acquires lock and iterates packs -> AssetRegistryLoader reads files -> AssetStores are populated

> Flow (Dynamic Pack Registration):
> Plugin calls **AssetModule.registerPack()** -> **AssetModule** dispatches AssetPackRegisterEvent -> Event handler acquires lock -> AssetRegistryLoader reads files from new pack -> AssetStores are updated with new assets

