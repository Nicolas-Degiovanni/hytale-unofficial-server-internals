---
description: Architectural reference for SpawningPlugin
---

# SpawningPlugin

**Package:** com.hypixel.hytale.server.spawning
**Type:** Singleton

## Definition
```java
// Signature
public class SpawningPlugin extends JavaPlugin {
```

## Architecture & Concepts
The SpawningPlugin is the central nervous system for all NPC spawning logic on the Hytale server. As a core `JavaPlugin`, it is instantiated once at server startup and orchestrates the entire lifecycle of NPC creation, population management, and despawning. It is not a class that is frequently interacted with directly during gameplay; rather, its primary role is to bootstrap the spawning subsystem by registering all necessary components, systems, and resources into the server's Entity Component System (ECS) framework.

This plugin is responsible for three primary types of spawning:
1.  **World Spawning:** Ambient, procedural spawning of NPCs in the world based on environmental factors like biome, time of day, and weather. This is managed by the `WorldSpawnManager`.
2.  **Beacon Spawning:** Point-of-interest spawning, where specific entities (Spawn Beacons) act as configurable hotspots for creating NPCs. This is managed by the `BeaconSpawnManager`.
3.  **Marker Spawning:** Spawning triggered by `SpawnMarker` entities, often placed by designers or world generation to create specific encounters.

During its `setup` phase, the plugin performs a massive number of registrations with the `EntityStoreRegistry` and `ChunkStoreRegistry`. This includes defining data components (e.g., `SpawnJobData`, `LocalSpawnController`), resources (e.g., `WorldSpawnData`, `SpatialResource`), and the systems that operate on them (e.g., `WorldSpawningSystem`, `SpawnBeaconSystems`). It also registers all spawn-related asset types, ensuring that configurations for spawns, markers, and suppression zones are loaded from disk.

The plugin acts as a reactive hub, listening for asset loading events (`LoadedAssetsEvent`, `RemovedAssetsEvent`). When spawn configurations are updated and reloaded, the plugin's event handlers trigger the appropriate managers to rebuild their internal data structures, allowing for dynamic, live updates to spawning behavior without a server restart.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's core plugin loader during the bootstrap sequence. A `JavaPluginInit` context is provided by the loader, giving it access to server registries. The static singleton instance is set within the `setup` method.
- **Scope:** The SpawningPlugin is a global singleton that persists for the entire server session. Its lifecycle is tied directly to the server's lifecycle.
- **Destruction:** The `shutdown` method is invoked by the plugin loader during the server shutdown process. Currently, this method is empty, as cleanup is managed by the garbage collector and the shutdown of the ECS registries themselves.

## Internal State & Concurrency
- **State:** The plugin maintains a significant amount of mutable state. This includes references to its subordinate managers (`WorldSpawnManager`, `BeaconSpawnManager`), a parsed configuration object (`config`), and handles for every component and resource type it registers. The managers themselves maintain complex, mutable caches of spawn configurations (`SpawnWrapper` objects) derived from loaded assets.

- **Thread Safety:** This class is **not thread-safe**. Its lifecycle methods (`setup`, `start`) and event handlers are designed to be called sequentially by the main server thread or a dedicated event bus thread. The systems it registers are executed by their respective world schedulers, which operate within a well-defined, single-threaded context per world tick.

    **WARNING:** Direct, concurrent modification of the plugin's internal state or its managers from other threads will lead to undefined behavior, race conditions, and server instability. All interaction with the spawning subsystem from other threads must be done through the thread-safe mechanisms of the ECS, such as command buffers.

## API Surface
The public API primarily consists of accessors for the ECS component and resource types it registers. These are used by other systems to query for spawning-related data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | SpawningPlugin | O(1) | Provides static access to the singleton instance. |
| setup() | void | O(N) | **Lifecycle Method.** Registers all spawning systems, components, resources, and assets. Must only be called by the plugin loader. |
| start() | void | O(1) | **Lifecycle Method.** Finalizes initialization by loading configuration values and caching required models. |
| getSpawnMarkerComponentType() | ComponentType | O(1) | Returns the handle for the SpawnMarkerEntity component. |
| getWorldSpawnDataResourceType() | ResourceType | O(1) | Returns the handle for the global WorldSpawnData resource. |
| getSpawnSuppressionControllerResourceType() | ResourceType | O(1) | Returns the handle for the SpawnSuppressionController resource. |
| shouldNPCDespawn(...) | boolean | O(1) | Core logic to determine if an NPC should be removed due to despawn rules or overpopulation. |
| validateSpawnsConfigurations(...) | void | O(N) | Static utility to validate that NPC roles referenced in spawn configs actually exist. |

## Integration Patterns

### Standard Usage
Developers and other systems do not typically invoke methods on the `SpawningPlugin` instance directly. Instead, they interact with the ECS constructs that the plugin has registered. For example, to create a new spawn point, one would create an entity and add the `SpawnBeacon` component to it. The `SpawnBeaconSystems` (registered by this plugin) will automatically detect and manage the new entity.

```java
// Example: Creating a new Spawn Beacon entity.
// The SpawningPlugin's systems will handle the rest.

// Obtain the component type registered by the plugin
ComponentType<EntityStore, SpawnBeacon> beaconType = EntityModule.get().getComponentType(SpawnBeacon.class);

// In a system, create the entity via a command buffer
commandBuffer.createEntity(
    new TransformComponent(position),
    new SpawnBeacon("some_beacon_config_id")
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SpawningPlugin()`. The server's plugin loader is solely responsible for its creation. Always use the static `SpawningPlugin.get()` method for access.
- **Manual Lifecycle Calls:** Do not call `setup()` or `start()` manually. These are strictly controlled by the server's lifecycle management.
- **External State Modification:** Do not directly access and modify the internal `worldSpawnManager` or `beaconSpawnManager`. Their state is managed internally based on asset loading events. Altering them externally will desynchronize the spawning system.

## Data Pipeline
The plugin establishes data flows by registering systems that react to events and component changes.

**Asset Loading & Validation Pipeline:**
> Flow:
> Server Startup -> Asset Loader discovers JSON files -> `LoadedAssetsEvent<WorldNPCSpawn>` -> **SpawningPlugin.onWorldNPCSpawnsLoaded** -> `worldSpawnManager.rebuildConfigurations()` -> Spawning rules are updated in memory.

**World (Ambient) Spawning Pipeline:**
> Flow:
> World Tick -> `WorldSpawningSystem` (registered by plugin) -> Scans loaded chunks -> Creates `SpawnJobData` component on eligible chunks -> `WorldSpawnJobSystems.Ticking` (registered by plugin) -> Processes job, performs flood-fill to find valid spawn location -> Creates `NPCEntity` via command buffer.

**Spawn Suppression Pipeline:**
> Flow:
> Entity with `SpawnSuppressionComponent` is created -> `SpawnSuppressionSystems.Suppressor` (registered by plugin) -> Detects nearby `SpawnMarkerEntity` instances using the `spawnMarkerSpatialResource` -> Adds suppression flags to the markers, preventing them from spawning.

