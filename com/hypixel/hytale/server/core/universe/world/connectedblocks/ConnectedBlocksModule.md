---
description: Architectural reference for ConnectedBlocksModule
---

# ConnectedBlocksModule

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks
**Type:** Singleton

## Definition
```java
// Signature
public class ConnectedBlocksModule extends JavaPlugin {
```

## Architecture & Concepts
The ConnectedBlocksModule is a core server plugin responsible for bootstrapping the "connected blocks" system. This system allows blocks to dynamically change their model or texture based on adjacent blocks, enabling features like connected fences, auto-cornering stairs, and complex roof structures.

This module acts as a central configuration and registration hub. It does not contain real-time game logic itself. Instead, its primary role is to integrate connected block functionality into the server's asset and event lifecycles. It achieves this by:
1.  **Registering Asset Types:** It informs the HytaleAssetStore how to recognize and decode custom asset files related to connected blocks, specifically the CustomConnectedBlockTemplateAsset.
2.  **Registering Codecs:** It registers the serializers and deserializers for different types of ConnectedBlockRuleSet (e.g., Stair, Roof, CustomTemplate), allowing these rules to be defined in data files and loaded by the engine.
3.  **Processing Loaded Assets:** It subscribes to the LoadedAssetsEvent for BlockType assets. After block definitions are loaded from disk, this module iterates through them, pre-processing and caching the connected block rules for efficient lookup during gameplay.

This module is a foundational piece of the world rendering and block-state system, ensuring that the rules governing block connectivity are loaded and applied before the world is simulated.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's core plugin loader during the server startup sequence. The framework provides a JavaPluginInit context to its constructor. The static singleton instance is set at this time.
-   **Scope:** The ConnectedBlocksModule is a session-scoped singleton. It persists for the entire runtime of the server.
-   **Destruction:** The object is destroyed and garbage collected when the server shuts down and the plugin system is unloaded.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its primary field, *instance*, is a static final reference set once during construction. The module's purpose is to modify the state of *external* systems, such as the global AssetRegistry and the in-memory representation of BlockType assets. It does not maintain its own mutable state across method calls.
-   **Thread Safety:** The module is thread-safe under normal operating conditions.
    -   The *setup* method is invoked by the plugin loader on the main server thread during initialization, which is a single-threaded phase.
    -   The *onBlockTypesChanged* event handler is static. It is invoked by the EventRegistry. **WARNING:** The safety of this handler depends on the engine's guarantee that asset loading events are dispatched synchronously on the main thread or within a memory-fenced context. If assets were reloaded asynchronously while the world is being accessed, this could lead to race conditions. The implementation assumes that the BlockTypeAssetMap is not being read by game logic threads while this event handler is executing.

## API Surface
The public API is minimal, as the module's primary function is configuration through side effects during its setup phase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | ConnectedBlocksModule | O(1) | Retrieves the global singleton instance of the module. |

## Integration Patterns

### Standard Usage
Direct interaction with this module is rare. Its functionality is implicitly consumed by other engine systems (like world generation and block placement) that use the processed BlockType assets. The primary integration is declarative, through the plugin manifest system.

A system needing to force a re-evaluation of block rules might (in a hypothetical scenario) retrieve the module like this:

```java
// This is an illustrative example; direct interaction is not a common pattern.
ConnectedBlocksModule module = ConnectedBlocksModule.get();
// No public methods exist to trigger its logic; it is entirely event-driven.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ConnectedBlocksModule()`. The server's plugin framework is solely responsible for its creation. Manual instantiation will break the singleton pattern and cause unpredictable behavior in the asset system.
-   **Manual Setup:** Do not invoke the *setup* method directly. It is a lifecycle method managed by the plugin loader and is not designed to be called more than once.

## Data Pipeline
The module operates as a processor within the server's asset loading pipeline. It transforms declarative asset data into cached, runtime-ready configuration.

> Flow:
> Block Asset Files (.json) -> HytaleAssetStore -> LoadedAssetsEvent -> **ConnectedBlocksModule::onBlockTypesChanged** -> In-Memory BlockTypeAssetMap (with cached rules) -> World Engine & Block Placement Logic

---

