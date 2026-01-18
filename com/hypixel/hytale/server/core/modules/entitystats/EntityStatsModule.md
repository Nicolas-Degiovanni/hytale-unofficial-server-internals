---
description: Architectural reference for EntityStatsModule
---

# EntityStatsModule

**Package:** com.hypixel.hytale.server.core.modules.entitystats
**Type:** Singleton

## Definition
```java
// Signature
public class EntityStatsModule extends JavaPlugin {
```

## Architecture & Concepts
The EntityStatsModule is a foundational server-side plugin that establishes the entire entity statistics system. It acts as the central bootstrap and registration authority for all concepts related to entity attributes like health, mana, stamina, and other gameplay-defined stats.

This module's primary responsibility is to integrate the entity statistics system with the server's core Asset and Entity-Component-System (ECS) frameworks. It does not manage the state of individual entities directly; rather, it sets up the necessary components, systems, and asset loaders that collectively form the stat system.

Key functions include:
-   **Asset Registration:** It registers the `EntityStatType` asset, allowing stats to be defined in JSON files. This decouples game design from engine code.
-   **Codec Registration:** It registers serialization codecs for `Modifier` and `Condition` objects. This enables complex, data-driven stat modification logic (e.g., "gain +10 strength while wielding an axe and health is below 50%").
-   **ECS Integration:** It registers the `EntityStatMap` component, which is attached to entities to store their live stat values. It also registers a suite of `EntityStatsSystems` that perform the logic for stat calculation, regeneration, and network synchronization within the ECS loop.
-   **Lifecycle Management:** It listens for asset loading events to trigger recalculations across all relevant entities when underlying stat definitions, items, or effects change.

In essence, this module is the architectural glue that connects data-driven stat definitions to the live, running game world.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's `JavaPlugin` loader during the server bootstrap sequence. The static singleton instance is set within the constructor, making it globally accessible via the `get()` method.
-   **Scope:** The singleton instance persists for the entire runtime of the game server. Its registered components and systems are active in every world managed by the server.
-   **Destruction:** The module is unloaded and its resources are implicitly released when the server shuts down.

## Internal State & Concurrency
-   **State:** The module itself is effectively stateless after its initial setup phase. It holds references to ECS `ComponentType` and `SystemType` handles, which are initialized once in the `setup` method and then treated as immutable. The actual state of entity statistics is stored decentrally in `EntityStatMap` components within the ECS `Store`.
-   **Thread Safety:** The module is not thread-safe during initialization. The `setup` and `start` methods are designed to be called sequentially by the main server thread. Post-initialization, the module is safe for concurrent access.
    -   The static `get()` method is thread-safe.
    -   The static `resolveEntityStats` helper methods are pure functions and are inherently thread-safe.
    -   Event handlers like `onLoadedAssetsEvent` follow a critical concurrency pattern: they receive an event (potentially from a worker thread) and immediately schedule the consequential logic to run on the appropriate world's thread using `world.execute()`. This prevents race conditions and ensures all ECS modifications are properly synchronized.

## API Surface
The public API is minimal, exposing only the singleton accessor and type handles necessary for other systems to interact with the stat framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | EntityStatsModule | O(1) | Provides global access to the singleton instance. |
| resolveEntityStats(raw) | Int2FloatMap | O(N) | Translates a map of string-based stat names to an integer-indexed map for high-performance lookups. |
| getEntityStatMapComponentType() | ComponentType | O(1) | Returns the ECS handle for the `EntityStatMap` component. |
| getStatModifyingSystemType() | SystemType | O(1) | Returns the ECS handle for the `StatModifyingSystem`. |

## Integration Patterns

### Standard Usage
Developers typically interact with the components and data structures registered by this module, rather than the module itself. The primary pattern is to retrieve the `EntityStatMap` from an entity to read or modify its stats.

```java
// Retrieve the stat map from an entity to check its health
EntityStatsModule statsModule = EntityStatsModule.get();
ComponentType<EntityStore, EntityStatMap> statMapType = statsModule.getEntityStatMapComponentType();

// Assume 'entityRef' is a valid reference to an entity
EntityStatMap stats = entityRef.getStore().getComponent(entityRef, statMapType);

if (stats != null) {
    float currentHealth = stats.getCurrent(DefaultEntityStatTypes.getHealth());
    float maxHealth = stats.getMaximum(DefaultEntityStatTypes.getHealth());
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new EntityStatsModule()`. The plugin loader is the sole authority for its creation. Using the constructor directly will break the singleton pattern and lead to an uninitialized, non-functional module.
-   **Premature Access:** Do not call `EntityStatsModule.get()` before the server's plugin loading phase is complete. Doing so will result in a `NullPointerException`.
-   **State Mutation:** Do not attempt to call registration methods like `getEntityStoreRegistry().registerSystem()` after the `setup` phase. The server's internal registries are finalized after initialization, and late registrations can cause unpredictable behavior or crashes.

## Data Pipeline
The module is central to two key data flows: asset definition to live data, and the in-game stat calculation loop.

**1. Asset Definition Pipeline**
This flow describes how a stat defined in a JSON file becomes a usable type in the game engine.

> Flow:
> `health.json` -> AssetStore Loader -> **`EntityStatsModule` (AssetType Registration)** -> `AssetRegistry` -> `LoadedAssetsEvent` -> **`EntityStatsModule` (Event Handler)** -> `DefaultEntityStatTypes.update()`

**2. In-Game Stat Calculation Pipeline**
This flow describes how an entity's stats are recalculated each tick based on modifiers from equipment, status effects, etc.

> Flow:
> Game Tick -> `EntityStatsSystems.Recalculate` -> Reads `StatModifiersManager` -> Calculates final values -> Writes to `EntityStatMap` component -> `EntityStatsSystems.Changes` -> Network Packet for client sync

