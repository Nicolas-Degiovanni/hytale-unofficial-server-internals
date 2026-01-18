---
description: Architectural reference for MacroCommandPlugin
---

# MacroCommandPlugin

**Package:** com.hypixel.hytale.builtin.commandmacro
**Type:** Singleton

## Definition
```java
// Signature
public class MacroCommandPlugin extends JavaPlugin {
```

## Architecture & Concepts
The MacroCommandPlugin is a core, built-in server plugin that acts as a bridge between the Hytale Asset System and the Command System. Its primary function is to enable the dynamic creation of server commands from data files, rather than from compiled Java code. This allows server operators and content creators to define complex, multi-step command sequences, or "macros", as simple assets.

The plugin introduces a new asset type, represented by the MacroCommandBuilder class. During server startup, it registers this asset type with the global AssetRegistry, instructing the engine to look for and load files from the `MacroCommands` asset path.

Upon a successful asset load or reload, the AssetRegistry fires a LoadedAssetsEvent. The MacroCommandPlugin subscribes to this event. Its event handler, loadCommandMacroAsset, iterates through the newly loaded macro definitions. For each definition, it dynamically constructs and registers a new command with the server's CommandRegistry. This process includes handling updates: if a macro asset is modified and reloaded, the plugin first unregisters the old command before registering the new version, ensuring seamless hot-reloading of command logic.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's internal PluginLoader during the server bootstrap sequence. The constructor receives a JavaPluginInit context object, which provides access to essential server systems like the EventRegistry and CommandRegistry.
- **Scope:** The plugin is a singleton that persists for the entire server session. The static `instance` field is set within the `setup` method, which is invoked by the PluginLoader immediately after construction.
- **Destruction:** The plugin is destroyed during the server shutdown process. While not explicitly shown, the plugin framework is responsible for ensuring that any registered commands and event listeners are properly cleaned up to prevent memory leaks.

## Internal State & Concurrency
- **State:** The plugin maintains a mutable internal state via the `macroCommandRegistrations` map. This map holds a reference to every CommandRegistration created from an asset. This state is critical for the hot-reloading feature, as it allows the plugin to find and unregister the previous version of a command when its source asset is updated.
- **Thread Safety:** This class is **not thread-safe**. All interactions with its methods and state are expected to occur on the main server thread. The `setup` method is called once during startup, and the `loadCommandMacroAsset` event handler is dispatched by the EventRegistry, which typically executes listeners on the main thread to prevent race conditions with game state. Direct, concurrent access from other threads will lead to undefined behavior.

## API Surface
The primary interaction with this plugin is through the creation of assets, not direct API calls.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static MacroCommandPlugin | O(1) | Retrieves the singleton instance of the plugin. |
| loadCommandMacroAsset(event) | void | O(N) | Event handler for processing loaded macro assets. N is the number of assets in the event. |

## Integration Patterns

### Standard Usage
The intended usage is entirely data-driven. A server administrator creates a command macro definition file (e.g., a JSON file) and places it in the appropriate server asset directory. The plugin handles the rest automatically.

Direct interaction is rare, but another plugin could retrieve the instance if necessary.

```java
// This is NOT a common use case.
// The plugin's functionality is automatic.
MacroCommandPlugin plugin = MacroCommandPlugin.get();
// There are no public methods to call for standard operations.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new MacroCommandPlugin()`. The server's PluginLoader is solely responsible for the lifecycle of plugins. Manual instantiation will result in a non-functional object that is not integrated with the server.
- **Static Access Before Setup:** Calling `MacroCommandPlugin.get()` before the server has completed its plugin loading phase will return null, leading to a NullPointerException.
- **State Manipulation:** Do not retrieve the plugin instance to directly modify the internal `macroCommandRegistrations` map. Doing so will corrupt the plugin's state and break the command hot-reloading mechanism.

## Data Pipeline
This plugin's core responsibility is to transform data on disk into executable in-game logic. The flow is unidirectional and event-driven.

> Flow:
> Macro Asset File on Disk -> AssetRegistry (File Watcher) -> Asset Deserialization (CODEC) -> LoadedAssetsEvent -> **MacroCommandPlugin::loadCommandMacroAsset** -> CommandRegistry::registerCommand -> Player can execute the new command

