---
description: Architectural reference for WorldGenPlugin
---

# WorldGenPlugin

**Package:** com.hypixel.hytale.builtin.worldgen
**Type:** Singleton

## Definition
```java
// Signature
public class WorldGenPlugin extends JavaPlugin {
```

## Architecture & Concepts
The WorldGenPlugin serves as a critical bootstrap component for Hytale's default world generation system. It is not a system that performs ongoing work, but rather a **registrar** that injects core world generation providers and entity systems into the server's central registries during startup.

Its primary architectural function is to act as a modular entry point, discovered and loaded by the server's plugin framework. By existing, it ensures that the server is aware of the "Hytale" world generation type and has the necessary systems, like BiomeDataSystem, to manage world-specific data. This pattern decouples the core server engine from specific game implementations, allowing different world generation logic to be swapped in or out via the plugin system.

This class is the bridge between the generic plugin lifecycle and the concrete implementation of Hytale's world generator.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's internal Plugin Loader during the server bootstrap phase. The constructor receives a JavaPluginInit context object, which provides access to core server registries.
- **Scope:** Session-scoped. The singleton instance persists for the entire duration of the server session, from startup to shutdown. The static instance is set within the `setup` method.
- **Destruction:** The object is de-referenced and becomes eligible for garbage collection when the plugin is unloaded by the server, typically during a controlled shutdown.

## Internal State & Concurrency
- **State:** This class is effectively stateless after initialization. Its only internal state is the static `instance` reference to itself. Its primary purpose is to modify the state of *external* systems (the EntityStoreRegistry and IWorldGenProvider.CODEC) during the `setup` phase.
- **Thread Safety:** The `setup` method is invoked once by the Plugin Loader on the main server thread during startup, eliminating any initialization race conditions. Subsequent calls to the static `get` method are simple, non-blocking reads and are inherently thread-safe. The systems it registers, BiomeDataSystem and HytaleWorldGenProvider, are responsible for their own thread safety.

## API Surface
The public contract is minimal, as this class is designed for framework integration, not direct developer interaction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static WorldGenPlugin | O(1) | Retrieves the singleton instance of the plugin. |

## Integration Patterns

### Standard Usage
Developers do not typically interact with this class directly. Its presence in the server's plugin directory is sufficient to enable the default Hytale world generation. The primary integration is declarative, through world configuration files.

**Example World Configuration (e.g., world.json):**
```json
{
  "generator": {
    "type": "Hytale",
    "seed": 12345
  }
}
```
The server can resolve the "Hytale" type to the HytaleWorldGenProvider class because this plugin registered it.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WorldGenPlugin()`. The plugin lifecycle is strictly managed by the server's Plugin Loader. Attempting to create an instance manually will bypass initialization and result in a non-functional or unstable state.
- **Manual Setup:** Never invoke the `setup` method directly. This is a lifecycle callback and calling it more than once will cause fatal errors due to duplicate system and codec registrations.

## Data Pipeline
This class is not an active participant in a data pipeline; it is responsible for **constructing** the pipeline at server start.

> **Initialization Flow:**
> Server Startup -> Plugin Loader Discovers and Loads Jar -> **WorldGenPlugin.setup()** is invoked -> Registers BiomeDataSystem and HytaleWorldGenProvider -> Setup Complete

> **Resulting WorldGen Flow:**
> World Creation Request (type: "Hytale") -> IWorldGenProvider Codec -> Resolves to **HytaleWorldGenProvider** -> Chunk Generation -> Biome Data is managed by **BiomeDataSystem**

