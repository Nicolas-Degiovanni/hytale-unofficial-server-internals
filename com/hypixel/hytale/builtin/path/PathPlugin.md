---
description: Architectural reference for PathPlugin
---

# PathPlugin

**Package:** com.hypixel.hytale.builtin.path
**Type:** Singleton

## Definition
```java
// Signature
public class PathPlugin extends JavaPlugin {
```

## Architecture & Concepts
The PathPlugin is the central authority for the server-side entity pathing framework. It does not implement pathfinding algorithms directly; rather, it establishes the entire Entity-Component-System (ECS) infrastructure required to define, manage, store, and visualize paths within the game world.

As a `JavaPlugin`, it serves as a bootstrapper and service registry for path-related functionality. During server initialization, it injects a suite of specialized components, resources, and systems into the core engine. These constructs collectively manage two primary types of paths: world paths and prefab-local paths.

Its key responsibilities include:
- **ECS Registration:** Registering custom components like WorldPathBuilder, resources like WorldPathData, and the PatrolPathMarker entity type.
- **System Orchestration:** Injecting a series of ordered systems (e.g., PrefabPathSystems) that react to entity lifecycle events to maintain path data integrity. This includes systems for spatial indexing, nameplate management, and world generation hooks.
- **Command Binding:** Exposing in-game commands (WorldPathCommand, PrefabPathCommand) for developers and builders to interact with the pathing system.
- **Asset Management:** Loading and managing the visual representation (the Model) for path markers, including support for hot-reloading assets.
- **Inter-Plugin Dependency:** It explicitly depends on the BuilderToolsPlugin, demonstrating a layered architecture where higher-level tools build upon this core pathing framework.

This plugin acts as the single source of truth for retrieving the handles (e.g., ResourceType, ComponentType) necessary to interact with pathing data within other game systems.

### Lifecycle & Ownership
- **Creation:** The PathPlugin is instantiated once by the server's plugin loader during the bootstrap sequence. The core framework injects a JavaPluginInit context, granting it access to essential engine registries. The static singleton instance is set within the `setup` method.
- **Scope:** Session-scoped. The plugin instance persists for the entire lifetime of the server process. It is designed to be a permanent, foundational service.
- **Destruction:** The plugin is destroyed when the server shuts down or if it is explicitly unloaded by the plugin manager. The JavaPlugin base class handles the teardown, though no custom cleanup logic is overridden in this class.

## Internal State & Concurrency
- **State:** The plugin's direct state is minimal but critical. It holds immutable handles to the ECS types it registers (e.g., worldPathDataResourceType). Its primary mutable state is the `pathMarkerModel` field, which holds the resolved 3D model for path markers. This model can be dynamically updated at runtime if the underlying assets change, as handled by the `onModelsChanged` event listener.
- **Thread Safety:** The plugin object itself is not designed for concurrent modification. Its initialization methods (`setup`, `start`) are invoked serially by the main server thread. Public accessor methods like `getWorldPathDataResourceType` are thread-safe as they return immutable handles.

**WARNING:** All state mutations and interactions with the pathing data structures *must* occur through the ECS systems registered by this plugin. These systems are executed by the world's update loop, which enforces its own concurrency model (typically a single-threaded tick). Direct, multi-threaded modification of path data from outside the ECS loop will lead to race conditions and world corruption.

## API Surface
The public API is designed as a service locator, providing access to the ECS handles that govern the pathing system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | PathPlugin | O(1) | Provides static access to the singleton instance. Throws if accessed before `setup` completes. |
| getWorldPathDataResourceType() | ResourceType | O(1) | Returns the handle for the WorldPathData resource, which stores global path information. |
| getPrefabPathSpatialResource() | ResourceType | O(1) | Returns the handle for the spatial resource (KDTree) used to accelerate queries for path markers. |
| getWorldPathBuilderComponentType() | ComponentType | O(1) | Returns the handle for the WorldPathBuilder component. |
| getPathMarkerModel() | Model | O(1) | Returns the currently loaded visual model for path markers. May be null before `start` completes. |

## Integration Patterns

### Standard Usage
Interaction with the pathing system should always be performed via the singleton instance to retrieve resource or component handles, which are then used with the ECS `Store` or `CommandBuffer`.

```java
// In another ECS system or plugin...
PathPlugin pathPlugin = PathPlugin.get();

// Get the handle for the world path data
ResourceType<EntityStore, WorldPathData> pathDataType = pathPlugin.getWorldPathDataResourceType();

// In a system's handle method, access the resource for the current world
public void handle(@Nonnull Store<EntityStore> store, ...) {
    WorldPathData pathData = store.getResource(pathDataType);
    // Now operate on the path data for this world
    pathData.compactPrefabPath(...);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new PathPlugin()`. The server's plugin framework is solely responsible for its creation and lifecycle. Attempting to construct it manually will fail due to the missing `JavaPluginInit` dependency and will break the singleton pattern.
- **Premature State Access:** Do not call `getPathMarkerModel()` before the plugin's `start` method has been executed by the server. The model is loaded from configuration and assets during this phase and will be null beforehand, leading to a NullPointerException.
- **Bypassing ECS:** Do not attempt to modify path data outside of the registered ECS systems. The systems contain critical logic for maintaining data consistency, such as updating the spatial index and managing entity relationships.

## Data Pipeline
The plugin establishes reactive data flows through its registered ECS systems. A primary example is the handling of prefab pasting events.

> Flow:
> Builder Tool pastes a prefab -> Engine fires `PrefabPasteEvent` -> `BuilderToolsPlugin.PrefabPasteEventSystem` processes it -> **`PathPlugin.PrefabPasteEventSystem`** (with `Order.AFTER` dependency) receives the event -> System retrieves `WorldPathData` resource -> System calls `worldPathDataResource.compactPrefabPath()` -> Path data is optimized and cleaned up post-paste.

This demonstrates how the plugin ensures its data remains consistent by hooking into broader engine events and carefully ordering its system execution relative to other plugins.

