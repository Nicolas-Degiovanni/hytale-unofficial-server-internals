---
description: Architectural reference for NPCShopPlugin
---

# NPCShopPlugin

**Package:** com.hypixel.hytale.builtin.adventure.npcshop
**Type:** Singleton (Plugin-Scoped)

## Definition
```java
// Signature
public class NPCShopPlugin extends JavaPlugin {
```

## Architecture & Concepts
The **NPCShopPlugin** is a bootstrap component responsible for integrating custom NPC shop behaviors into the server's core Non-Player Character (NPC) system. It does not manage active game state or handle runtime logic. Instead, its primary function is to act as a registration bridge during server initialization.

This plugin extends the functionality of the central **NPCPlugin** service by registering two new component types: **OpenShop** and **OpenBarterShop**. It maps these string identifiers to their corresponding Java implementation classes, **BuilderActionOpenShop** and **BuilderActionOpenBarterShop**, respectively.

This registration allows game designers and content creators to define shop-opening behaviors for NPCs declaratively in data files (e.g., JSON or HOCON). When the server later parses an NPC definition containing an **OpenShop** component, the **NPCPlugin** system uses the mapping established by this plugin to instantiate the correct action handler. In essence, this plugin translates a data-driven configuration into executable server-side logic.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's central **PluginLoader** during the server startup sequence. The loader injects a **JavaPluginInit** context object into the constructor, providing the plugin with its operational environment.
- **Scope:** The object instance persists for the entire server session. Its lifecycle is directly managed by the Hytale plugin framework.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down or the plugin is explicitly unloaded by an administrator. It has no explicit teardown logic.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no mutable fields and does not cache any data. Its sole purpose is to execute a one-time configuration task within its **setup** method. The state it modifies is external, residing within the singleton **NPCPlugin** registry.
- **Thread Safety:** The plugin framework guarantees that the **setup** method is invoked from the main server thread during the initialization phase. As such, the class is not designed for concurrent access and requires no internal synchronization. All operations are thread-safe by virtue of the managed, single-threaded initialization environment.

## API Surface
The public contract of this class is defined by its role as a **JavaPlugin**, which is primarily consumed by the plugin loader. It exposes no API for direct use by other game systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | protected void | O(1) | Framework-invoked lifecycle method. Registers the **OpenShop** and **OpenBarterShop** component types. **WARNING:** Direct invocation will lead to system instability. |

## Integration Patterns

### Standard Usage
This class is not used directly in code. It is enabled by including it in the server's plugin configuration. Its functionality is consumed declaratively by content creators when defining NPC behaviors.

The following conceptual example shows an NPC definition that relies on this plugin's registrations.

```yaml
# Example: my_shopkeeper.npc.json
# This file is only valid after the NPCShopPlugin has been loaded.

components: {
  "hytale:name": "Shopkeeper Steve",
  "hytale:interact_action": {
    "type": "OpenShop", # This key is registered by NPCShopPlugin
    "shopId": "village_general_store"
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate this class with **new NPCShopPlugin()**. The plugin framework is solely responsible for its creation and lifecycle management. Attempting to do so will fail due to the missing **JavaPluginInit** context.
- **Manual Invocation:** Do not call the **setup** method manually. This method is part of a managed lifecycle and is designed to be called only once by the plugin loader. Calling it again will cause the **NPCPlugin** to throw an exception for duplicate component registration.

## Data Pipeline
This plugin operates during the server's initialization pipeline, not a runtime data processing pipeline. Its output is a modification to a central registry, which subsequently affects how NPC data files are parsed.

> **Initialization Flow:**
> Server Start → Plugin Loader Discovers **NPCShopPlugin** → **NPCShopPlugin.setup()** is invoked → Component types are written to **NPCPlugin** Registry → Server proceeds to load NPC definition files, which can now be parsed with the newly registered components.

