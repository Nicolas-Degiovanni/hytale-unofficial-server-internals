---
description: Architectural reference for FarmingPlugin
---

# FarmingPlugin

**Package:** com.hypixel.hytale.builtin.adventure.farming
**Type:** Singleton

## Definition
```java
// Signature
public class FarmingPlugin extends JavaPlugin {
```

## Architecture & Concepts
The FarmingPlugin serves as the central bootstrap and registration authority for the entire server-side farming system. It is not a system that processes data during the game loop; rather, its primary responsibility is to initialize and integrate all farming-related components, assets, interactions, and logic into the core server engine during startup.

It acts as a module definition, declaring all the constituent parts of the farming feature:
*   **Asset Registration:** It registers custom asset types like FarmingCoopAsset and GrowthModifierAsset, making them known to the HytaleAssetStore.
*   **Component Registration:** It defines and registers the fundamental data components for the Entity-Component-System (ECS) framework, such as TilledSoilBlock, FarmingBlock, and CoopResidentComponent. These components attach state to blocks and entities in the world.
*   **System Registration:** It registers the active logic (the "S" in ECS) that operates on farming components, such as FarmingSystems.Ticking, which is responsible for crop growth over time.
*   **Codec and Interaction Registration:** It registers custom data serializers (codecs) and player interactions like HarvestCropInteraction and FertilizeSoilInteraction. This allows the server to understand farming-related data formats and player actions.
*   **Event Handling:** It subscribes to core engine events, such as ChunkPreLoadProcessEvent, to inject specialized logic at critical points in the world lifecycle, like preventing crop spread in newly generated chunks.

In essence, FarmingPlugin wires the farming module into the engine's registries, ensuring the core systems know how to load, simulate, and interact with all farming-related objects.

## Lifecycle & Ownership
- **Creation:** A single instance of FarmingPlugin is created by the server's core plugin loader during the server bootstrap sequence. The constructor receives a JavaPluginInit object, which provides access to the engine's core registries.
- **Scope:** The instance is a singleton that persists for the entire server session. The static *instance* field is set within the *setup* method, making it globally accessible via the static *get* method.
- **Destruction:** The object is dereferenced and eligible for garbage collection when the server shuts down or if the plugin is unloaded by the plugin manager.

## Internal State & Concurrency
- **State:** The internal state of the FarmingPlugin consists primarily of ComponentType handles (e.g., tiledSoilBlockComponentType). These fields are initialized once during the *setup* method and are considered immutable for the remainder of the server's lifecycle. They act as cached, read-only identifiers for querying the ECS registries.

- **Thread Safety:** This class is **conditionally thread-safe**.
    - The *setup* method is executed by the main server thread during the single-threaded initialization phase. It is not safe to call from any other thread or at any other time.
    - After initialization, the ComponentType fields are effectively final and can be safely read from any thread.
    - The static *preventSpreadOnNew* event handler is invoked by the world generation system. Assume this occurs on a world-specific thread. The operations within are on chunk data, and thread safety is therefore delegated to the underlying world and chunk management systems.

    **Warning:** Do not assume the event handler can safely access arbitrary global state. It should only operate on the data provided by the event object.

## API Surface
The public API is minimal, designed to provide other systems with the necessary handles to interact with the farming ECS components.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | FarmingPlugin | O(1) | Retrieves the global singleton instance of the plugin. |
| getTiledSoilBlockComponentType() | ComponentType | O(1) | Returns the registered type handle for the TilledSoilBlock component. |
| getFarmingBlockComponentType() | ComponentType | O(1) | Returns the registered type handle for the FarmingBlock component. |
| getFarmingBlockStateComponentType() | ComponentType | O(1) | Returns the registered type handle for the FarmingBlockState component. |
| getCoopBlockStateComponentType() | ComponentType | O(1) | Returns the registered type handle for the CoopBlock component. |
| getCoopResidentComponentType() | ComponentType | O(1) | Returns the registered type handle for the CoopResidentComponent. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to retrieve the singleton instance and then fetch the ComponentType handles. These handles are then used with the ECS API to get or set farming data on world objects.

```java
// Retrieve the component type for farming blocks
ComponentType<ChunkStore, FarmingBlock> farmingType = FarmingPlugin.get().getFarmingBlockComponentType();

// Use the type to get the component from a block entity holder
Holder<ChunkStore> blockEntity = world.getBlockEntityAt(position);
FarmingBlock farmingComponent = blockEntity.getComponent(farmingType);

if (farmingComponent != null) {
    // ... process farming logic
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call *new FarmingPlugin()*. The plugin must be instantiated by the server's plugin loader to be correctly initialized with the engine's registries. Direct instantiation will result in a non-functional object.
- **Premature Access:** Do not attempt to call *FarmingPlugin.get()* before the server's plugin loading phase is complete. This can lead to a NullPointerException if the *instance* field has not yet been set. The engine's lifecycle management typically prevents this.

## Data Pipeline
FarmingPlugin does not participate in a runtime data pipeline. Instead, its role is to **construct** the pipelines that will run later. It injects logic and data definitions into various engine systems.

> **Setup-Time Flow:**
> **FarmingPlugin.setup()** -> Registers handlers with ->
> 1.  **AssetRegistry**: Defines how to load *.json* files for GrowthModifierAsset.
> 2.  **ChunkStoreRegistry**: Registers FarmingBlock components and FarmingSystems.Ticking.
> 3.  **EntityStoreRegistry**: Registers CoopResidentComponent and its associated systems.
> 4.  **CodecRegistry**: Defines how to serialize and deserialize interactions like HarvestCropInteraction.

This setup enables multiple runtime data flows. For example, the crop growth pipeline:

> **Runtime Crop Growth Flow:**
> Game Tick -> ChunkStoreRegistry -> **FarmingSystems.Ticking** -> Reads/Writes **FarmingBlockState** Component -> World State Change

