---
description: Architectural reference for BlockPhysicsPlugin
---

# BlockPhysicsPlugin

**Package:** com.hypixel.hytale.builtin.blockphysics
**Type:** Transient

## Definition
```java
// Signature
public class BlockPhysicsPlugin extends JavaPlugin {
```

## Architecture & Concepts
The BlockPhysicsPlugin class serves as the primary bootstrap and integration point for the server-side block physics systems. It is not the engine of block physics itself; rather, it is a lightweight module responsible for injecting the necessary physics logic into the server's core lifecycle at two critical stages: startup and the main game loop.

Its architectural role is to act as a declarative bridge:
1.  **Startup Integration:** It registers a validation routine that runs during the asset loading phase. This ensures that all block-related prefabs conform to the required data contracts before the server becomes available, preventing runtime errors due to malformed assets.
2.  **Game Loop Integration:** It registers a persistent, ticking system with the ChunkStoreRegistry. This registered system, BlockPhysicsSystems.Ticking, contains the actual logic that is executed each game tick to simulate physics phenomena like gravity or structural integrity for blocks.

This class follows a standard plugin pattern, where the server's plugin loader discovers, instantiates, and manages its lifecycle, calling the setup method at the appropriate time.

## Lifecycle & Ownership
- **Creation:** A single instance of BlockPhysicsPlugin is created by the server's plugin loading mechanism during the initial bootstrap sequence. The framework supplies a JavaPluginInit context object to its constructor, which provides access to core server registries.
- **Scope:** The instance persists for the entire server session. Its registrations with the EventRegistry and ChunkStoreRegistry also persist for the lifetime of the server.
- **Destruction:** The instance is dereferenced and eligible for garbage collection during the server shutdown process when all plugins are unloaded.

## Internal State & Concurrency
- **State:** This class is effectively stateless. It does not maintain or cache any mutable data. Its sole purpose is to perform one-time registration actions during its setup phase.
- **Thread Safety:** The setup method is guaranteed by the framework to be called once from the main server thread during initialization. The registered event handler, validatePrefabs, is invoked by the EventRegistry; assuming startup events are processed serially on the main thread, this operation is safe. The class is not designed for and should not be accessed from multiple threads.

## API Surface
The public contract is minimal and intended for framework use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | protected void | O(1) | Called by the plugin framework. Registers the prefab validator and the ticking physics system. |
| validatePrefabs(event) | public static void | O(N) | Event handler for LoadAssetEvent. Triggers prefab validation. Complexity is proportional to the number of prefabs. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is automatically discovered and managed by the Hytale server's plugin system. Its behavior is controlled through server configuration, not direct API calls. To enable or disable its functionality, a server operator would modify the server's plugin manifest or configuration files.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new BlockPhysicsPlugin(). The class requires a framework-supplied JavaPluginInit context for its constructor. Attempting to create it manually will fail or result in a non-functional plugin.
- **Manual Invocation:** Do not call the setup method directly. The plugin loader is responsible for invoking this method at the correct point in the server's startup lifecycle to ensure registries are available.

## Data Pipeline
BlockPhysicsPlugin initiates two distinct control flows rather than processing a continuous stream of data.

**Flow 1: Startup Asset Validation**
> Server Bootstrap → Plugin Loader → **BlockPhysicsPlugin.setup()** → EventRegistry.register → LoadAssetEvent Fired → **BlockPhysicsPlugin.validatePrefabs()** → PrefabBufferValidator → Server Halt (on failure)

**Flow 2: Game Loop System Registration**
> Server Bootstrap → Plugin Loader → **BlockPhysicsPlugin.setup()** → ChunkStoreRegistry.registerSystem(TickingSystem) → Game Loop Tick → TickingSystem.update()

