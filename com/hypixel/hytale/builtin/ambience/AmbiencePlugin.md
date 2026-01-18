---
description: Architectural reference for AmbiencePlugin
---

# AmbiencePlugin

**Package:** com.hypixel.hytale.builtin.ambience
**Type:** Singleton

## Definition
```java
// Signature
public class AmbiencePlugin extends JavaPlugin {
```

## Architecture & Concepts
The AmbiencePlugin serves as the central bootstrap and registry for the server-side ambience and music subsystem. As a core `JavaPlugin`, it integrates directly with the server's startup lifecycle to configure and enable all features related to environmental audio.

Its primary architectural role is that of a **Module Initializer**. It does not contain any per-tick game logic itself. Instead, its responsibility is to register all the necessary Entity-Component-System (ECS) constructs with the engine's core registries. This includes:

*   **Components:** It defines the data structures for tracking ambient state, such as AmbienceTracker for players and AmbientEmitterComponent for world objects.
*   **Systems:** It registers the systems, like AmbientEmitterSystems and ForcedMusicSystems, which contain the actual logic for processing components each tick or in response to game events.
*   **Resources:** It registers world-scoped data, such as the AmbienceResource.
*   **Commands:** It exposes administrative commands via AmbienceCommands for debugging or control.

By centralizing these registrations, the plugin ensures that the entire ambience feature set is initialized atomically and correctly within the server's lifecycle. It also acts as a service locator for other parts of the ambience code, providing access to registered component types and configured assets.

## Lifecycle & Ownership
- **Creation:** The AmbiencePlugin is instantiated exclusively by the Hytale server's plugin management system during server boot. The constructor is invoked with a JavaPluginInit context, which provides access to the server's core registries. Immediately upon construction, it assigns its own reference to a static `instance` field, establishing the singleton pattern.

- **Scope:** The plugin instance is a server-scoped singleton. It persists for the entire runtime of the server process, from initial loading to final shutdown.

- **Destruction:** The object is decommissioned when the plugin is unloaded during a graceful server shutdown. All its registered systems and components are deregistered by the core engine as part of this process.

## Internal State & Concurrency
- **State:** The internal state of the AmbiencePlugin is minimal and primarily established during server initialization.
    - It holds references to the `ComponentType` and `ResourceType` handles returned by the engine's registries during the `setup` phase. These are assigned once and are immutable thereafter.
    - It manages a `Config` object, which provides a typed view over the plugin's configuration file.
    - It loads and caches the `Model` for the ambient emitter marker during the `start` phase.
    - **Warning:** The state is initialized across two distinct lifecycle methods, `setup` and `start`. State initialized in `start` (e.g., the ambientEmitterModel) will be null if accessed prematurely.

- **Thread Safety:** The class is thread-safe under the operational guarantees of the Hytale server.
    - The lifecycle methods `setup` and `start` are invoked serially by the main server thread during startup. All state initialization is therefore free of race conditions.
    - All public getter methods return references that were safely published during the single-threaded initialization phase. They can be safely called from any system running on any thread, as the values they return are effectively constant after server startup.

## API Surface
The public API is designed for service location, allowing other parts of the ambience system to retrieve foundational types and assets.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | AmbiencePlugin | O(1) | Statically retrieves the singleton instance of the plugin. |
| getAmbienceTrackerComponentType() | ComponentType | O(1) | Returns the engine handle for the AmbienceTracker component. |
| getAmbientEmitterComponentType() | ComponentType | O(1) | Returns the engine handle for the AmbientEmitterComponent. |
| getAmbienceResourceType() | ResourceType | O(1) | Returns the engine handle for the AmbienceResource. |
| getAmbientEmitterModel() | Model | O(1) | Returns the configured visual model for ambient emitters. **Warning:** Returns null if called before the `start` lifecycle phase. |

## Integration Patterns

### Standard Usage
The canonical use of this class is to retrieve the singleton instance to access its registered type handles. This is common within other systems or commands that need to interact with ambience components.

```java
// Example from within another system or command
AmbiencePlugin plugin = AmbiencePlugin.get();
ComponentType<EntityStore, AmbienceTracker> trackerType = plugin.getAmbienceTrackerComponentType();

// Now use the trackerType to query or modify components on entities
...
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new AmbiencePlugin()`. The server's plugin loader is the sole authority for creating plugin instances. Attempting to do so manually will break the server lifecycle and result in a non-functional, detached object.

- **Premature State Access:** Do not call `getAmbientEmitterModel` from the `setup` phase of another plugin or system. The model is loaded in the `start` phase, which occurs after all plugins have completed their `setup` phase. Accessing it too early will result in a NullPointerException.

## Data Pipeline
The AmbiencePlugin does not directly process data. Instead, it **constructs** the data processing pipelines by registering systems with the engine. The conceptual flow of data through the systems registered by this plugin is as follows:

> **Flow (Ambient Emitters):**
> Game Tick -> **AmbientEmitterSystems.Ticking** (Registered by Plugin) -> Reads `AmbientEmitterComponent` -> Calculates Audio Event -> Sends to Client Audio Engine

> **Flow (Player Music):**
> Player Joins World -> **ForcedMusicSystems.PlayerAdded** (Registered by Plugin) -> Attaches `AmbienceTracker` Component -> Game Tick -> **ForcedMusicSystems.Tick** (Registered by Plugin) -> Reads `AmbienceTracker` & World State -> Determines Music Track -> Sends to Client Audio Engine

