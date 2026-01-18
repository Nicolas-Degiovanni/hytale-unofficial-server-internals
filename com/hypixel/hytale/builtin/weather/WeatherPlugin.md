---
description: Architectural reference for WeatherPlugin
---

# WeatherPlugin

**Package:** com.hypixel.hytale.builtin.weather
**Type:** Singleton

## Definition
```java
// Signature
public class WeatherPlugin extends JavaPlugin {
```

## Architecture & Concepts
The WeatherPlugin serves as the central bootstrap and registry for the entire weather module. As a subclass of JavaPlugin, it is a self-contained, loadable unit discovered and managed by the server's core plugin system.

Its primary architectural responsibility is to integrate weather-related logic into the main server engine during the startup sequence. It achieves this by performing several key registrations:

*   **Component & Resource Registration:** It defines and registers the WeatherTracker component and WeatherResource with the global EntityStore registry. This makes weather-related data available within the server's Entity Component System (ECS) framework.
*   **System Registration:** It registers multiple WeatherSystem classes, each responsible for a specific piece of logic (e.g., handling player joins, world ticks). These systems are the "brains" of the weather simulation and are driven by the main game loop.
*   **Command Registration:** It registers the WeatherCommand, exposing administrative control over the weather via the server's command-line interface.

Architecturally, this class is a **Module Entry Point**. It does not contain any simulation logic itself; instead, it acts as a manifest and initializer that wires the weather module's components into the appropriate core engine systems. The static singleton accessor, WeatherPlugin.get(), provides a global service locator for other parts of the engine to retrieve weather-specific type definitions.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's core plugin loader during the server bootstrap phase. The constructor requires a JavaPluginInit context object, which is supplied by the loader. The static singleton instance is set within the constructor.
- **Scope:** Session-scoped. The WeatherPlugin instance is created once when the server starts and persists until the server shuts down or the plugin is unloaded.
- **Destruction:** Managed by the plugin loader. There is no explicit public destruction method; cleanup is handled when the plugin's ClassLoader is released by the server.

## Internal State & Concurrency
- **State:** The class holds mutable state in its fields (weatherTrackerComponentType, weatherResourceType). However, this state is only written to once during the single-threaded `setup` phase of the server lifecycle. After initialization, the state is effectively immutable and serves as a cache for the registered type handles.

- **Thread Safety:** This class is **not thread-safe** during initialization but is safe for read-only access after the `setup` method completes.
    - The constructor and `setup` method **must** be executed in a single-threaded context, which is guaranteed by the server's plugin loading sequence.
    - The static `instance` field is subject to race conditions if multiple threads were to load this plugin concurrently, but this is a scenario prevented by the plugin framework's design.
    - The public getter methods are safe to call from any thread *after* the plugin has been fully initialized. Calling them before `setup` completes will result in a NullPointerException.

## API Surface
The public API is minimal, designed to provide access to the registered type handles rather than to be interacted with directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static WeatherPlugin | O(1) | Returns the global singleton instance of the plugin. |
| getWeatherTrackerComponentType() | ComponentType | O(1) | Returns the registered type handle for the WeatherTracker component. **Warning:** Throws NullPointerException if called before setup. |
| getWeatherResourceType() | ResourceType | O(1) | Returns the registered type handle for the WeatherResource. **Warning:** Throws NullPointerException if called before setup. |

## Integration Patterns

### Standard Usage
The standard pattern is to use the static accessor to retrieve the singleton, then use that instance to get the component and resource types needed to interact with the ECS.

```java
// Retrieve the registered type for the WeatherTracker component
WeatherPlugin plugin = WeatherPlugin.get();
ComponentType<EntityStore, WeatherTracker> trackerType = plugin.getWeatherTrackerComponentType();

// Use the type to access component data on an entity
WeatherTracker weather = entity.get(trackerType);
if (weather != null) {
    weather.setRaining(true);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WeatherPlugin()`. The plugin framework is solely responsible for creating and managing the lifecycle of this object. Attempting to do so manually will fail due to the missing JavaPluginInit context and will break the singleton pattern.
- **Premature Access:** Do not call `getWeatherTrackerComponentType()` or `getWeatherResourceType()` from another plugin's constructor or initialization block without ensuring the WeatherPlugin has already been loaded and set up. This can create a race condition resulting in a NullPointerException.

## Data Pipeline
This class is not part of a runtime data pipeline. Instead, it is a **Configuration Pipeline** component that executes once at startup to build the systems that will later process data.

> **Setup Flow:**
> Server Bootstrap → Plugin Loader → **WeatherPlugin.setup()** → Registers Systems, Components, and Commands with Core Engine → Runtime Systems (e.g., WeatherSystem) are now active in the Game Loop

