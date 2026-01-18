---
description: Architectural reference for WorldLocationConditionPlugin
---

# WorldLocationConditionPlugin

**Package:** com.hypixel.hytale.builtin.adventure.worldlocationcondition
**Type:** Plugin EntryPoint

## Definition
```java
// Signature
public class WorldLocationConditionPlugin extends JavaPlugin {
```

## Architecture & Concepts
The WorldLocationConditionPlugin is a registration-only plugin that serves as a bridge between the server's plugin loading system and the world logic system. Its sole architectural function is to extend the set of available world location conditions by registering a new type, the NeighbourBlockTagsLocationCondition, with a central, static registry.

This class exemplifies the **Registry** pattern. During server initialization, the plugin is loaded and its setup method is invoked. This method populates a global codec with the information needed to serialize and deserialize a new type of condition. This allows world designers and content creators to define complex location-based rules in data files (e.g., JSON) using a simple string identifier, "NeighbourBlockTags", without needing to modify the core server engine. The engine can then use the registered codec to transform this data into a live, executable Java object.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's internal PluginLoader service during the server startup sequence or a plugin hot-reload event. The loader discovers this class via plugin metadata and provides the required JavaPluginInit context during construction.

- **Scope:** The object instance persists for as long as the plugin is considered "loaded" by the server. Its primary work is completed in a single, initial burst during the setup phase.

- **Destruction:** The instance is marked for garbage collection when the plugin is formally unloaded by the server, typically during a full server shutdown or a command that cycles the plugin system.

**Warning:** The registration performed by this plugin modifies a global, static `CODEC` object. This modification will persist for the entire lifetime of the server process, even if the plugin itself is unloaded.

## Internal State & Concurrency
- **State:** This class is effectively stateless. It does not maintain any mutable state after the `setup` method completes. The context provided to its constructor is used only for initialization of the parent JavaPlugin and is not retained.

- **Thread Safety:** The class instance itself is inherently thread-safe due to its stateless design. However, the core operation—writing to the static `WorldLocationCondition.CODEC`—is not an atomic or thread-safe operation on its own. The server's plugin framework guarantees safety by ensuring that all plugin `setup` methods are executed sequentially from a single thread during the server's bootstrap phase.

**Warning:** Any attempt to invoke the `setup` method outside of the framework-controlled lifecycle, especially from multiple threads, would create a severe race condition on the central codec registry, leading to corrupted server state.

## API Surface
The public contract is defined by the JavaPlugin superclass. The only significant behavior is the implementation of the `setup` lifecycle method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | protected void | O(1) | Registers the NeighbourBlockTagsLocationCondition with the global codec. This is a lifecycle callback managed by the plugin framework and must not be invoked directly. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is a self-contained unit of functionality that is automatically discovered and activated by the server's plugin system. The standard "usage" is to ensure the plugin is installed on the server.

The *result* of this plugin enables declarative use of the new condition in world data files, as shown in this conceptual example:

```json
// A conceptual data file (e.g., for a creature's spawn rule)
// that relies on this plugin having been loaded by the server.
{
  "mob": "hytale:crawler",
  "spawnRules": [
    {
      "condition": {
        "type": "NeighbourBlockTags",
        "tags": ["cave_wall", "stone"],
        "minCount": 2,
        "radius": 1
      }
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new WorldLocationConditionPlugin()`. The plugin requires a specific initialization context that is only provided by the server's plugin framework. Manual instantiation will result in a non-functional object.

- **Manual Invocation:** Do not call the `setup` method directly. It is a lifecycle callback designed to be invoked once by the plugin loader. Calling it manually can cause duplicate registration exceptions or overwrite existing codec data, leading to unpredictable server behavior.

## Data Pipeline
This plugin is an **initialization-time component**, not a part of a continuous data processing pipeline. It acts once to configure a system that is then used repeatedly by other data pipelines.

> **Phase 1: Registration (at Server Startup)**
> Server Bootstrap -> PluginLoader Discovers Plugin -> **WorldLocationConditionPlugin.setup()** -> Mutates Global `WorldLocationCondition.CODEC` Registry

> **Phase 2: Usage (during Gameplay/World-Gen)**
> World Data File (e.g., JSON) -> Server Deserializer -> Accesses `WorldLocationCondition.CODEC` -> Instantiates `NeighbourBlockTagsLocationCondition` Object -> World Logic System Executes Condition

