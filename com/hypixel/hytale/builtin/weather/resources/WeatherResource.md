---
description: Architectural reference for WeatherResource
---

# WeatherResource

**Package:** com.hypixel.hytale.builtin.weather.resources
**Type:** Transient Data Component

## Definition
```java
// Signature
public class WeatherResource implements Resource<EntityStore> {
```

## Architecture & Concepts
The WeatherResource is a pure **data component** that encapsulates the weather state for a specific game world or region. It is not a service and contains no simulation logic; its sole responsibility is to act as the authoritative state container for weather conditions.

This class is attached to an EntityStore, which typically represents a server-side world instance. It serves as the central point of state management that the primary WeatherSystem reads from and writes to. The design distinguishes between two fundamental weather control mechanisms:

1.  **Forced Weather:** A globally overridden weather state, typically triggered by an administrator command or a scripted event. This takes precedence over natural weather.
2.  **Environment Weather:** The naturally simulated weather, tracked on a per-environment (e.g., biome) basis. This is the default state when no forced weather is active.

By centralizing this state into a distinct Resource, the engine decouples the weather simulation logic from the state itself, allowing other systems to query weather conditions without needing a direct dependency on the simulation code.

## Lifecycle & Ownership
-   **Creation:** A WeatherResource is not instantiated directly. It is created on-demand by the engine's resource management system when first requested for an EntityStore. The static `getResourceType` method provides the key for this lookup and creation process.
-   **Scope:** The lifecycle of a WeatherResource is strictly bound to its parent EntityStore. It persists as long as the world it represents is loaded in memory.
-   **Destruction:** The object is marked for garbage collection when its parent EntityStore is unloaded from the server. There are no manual cleanup methods.

## Internal State & Concurrency
-   **State:** This component is highly **mutable**. Its fields are continuously updated by the server's weather simulation systems and command handlers to reflect the dynamic state of the world's weather. It caches the current weather index for each environment and tracks state changes for forced weather and game time.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. The state-consuming methods like `consumeForcedWeatherChange` and `compareAndSwapHour` are particularly vulnerable to race conditions if accessed concurrently.

    **WARNING:** All interactions with a WeatherResource instance must be performed from a single, synchronized context, typically the main server game thread responsible for its parent EntityStore. Unmanaged multi-threaded access will lead to state corruption and server instability.

## API Surface
The public API is designed for state management, providing methods to set, query, and consume state changes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setForcedWeather(String) | void | O(log N) | Overrides natural weather. Looks up the weather name in the global asset map. A null value disables forced weather. |
| consumeForcedWeatherChange() | boolean | O(1) | A state-consuming check. Returns true if the forced weather has changed since the last call, then resets its internal change tracker. |
| compareAndSwapHour(int) | boolean | O(1) | A state-consuming check. Returns true if the provided hour is different from the last recorded hour, then updates the internal state. |
| getWeatherIndexForEnvironment(int) | int | O(1) | Retrieves the current natural weather index for a specific environment ID. |
| getEnvironmentWeather() | Int2IntMap | O(1) | Returns the raw map of environment IDs to weather indices. Use with caution, as direct modification bypasses any logic. |

## Integration Patterns

### Standard Usage
The WeatherResource is managed by a higher-level system, such as a WeatherSystem, which executes on the main game tick. This system retrieves the resource from the world, checks for state changes, runs simulations, and updates the resource accordingly.

```java
// Example from a hypothetical WeatherSystem update tick
EntityStore worldStore = world.getEntityStore();
WeatherResource weatherState = worldStore.getResource(WeatherResource.getResourceType());

// Check for admin commands
if (weatherState.consumeForcedWeatherChange()) {
    // A command like /weather has been used.
    // Propagate this new forced state to all players.
    broadcastWeatherUpdate(weatherState.getForcedWeatherIndex());
    return; // Skip natural simulation
}

// Check if a new hour has begun to trigger natural weather changes
if (weatherState.compareAndSwapHour(world.getTime().getCurrentHour())) {
    // Run simulation logic for each environment
    for (int environmentId : world.getActiveEnvironments()) {
        int newWeather = calculateNaturalWeatherFor(environmentId);
        weatherState.getEnvironmentWeather().put(environmentId, newWeather);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new WeatherResource()`. The engine's resource system is responsible for the object's lifecycle. Manually creating an instance will result in a disconnected state object that the engine is unaware of.
-   **Asynchronous Modification:** Do not modify the WeatherResource from a separate thread or asynchronous task. For example, a network packet handler should not directly call `setForcedWeather`. It should instead dispatch a task or event to be executed on the main game thread.
-   **State Hoarding:** Do not cache the result of `getEnvironmentWeather()` across multiple game ticks. The map can be changed at any time by the simulation system, and holding a stale reference will lead to incorrect logic. Fetch the resource from the EntityStore at the start of each logical operation.

## Data Pipeline
The WeatherResource acts as a central state repository in the weather data flow. It does not process data itself but is the location where data is stored and retrieved.

**Input Flow (Writing State):**
> Game Command (`/weather`) -> Command System -> **WeatherResource**.setForcedWeather()
>
> Server Tick -> Weather Simulation System -> **WeatherResource**.getEnvironmentWeather().put()

**Output Flow (Reading State):**
> **WeatherResource** -> Weather Propagation System -> Network Packet -> Client-side Weather Rendering
>
> **WeatherResource** -> AI Behavior System -> Query Weather -> Modify NPC Behavior

