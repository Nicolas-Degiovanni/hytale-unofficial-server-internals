---
description: Architectural reference for WeatherSystem
---

# WeatherSystem

**Package:** com.hypixel.hytale.builtin.weather.systems
**Type:** System Container

## Definition
```java
// Signature
public class WeatherSystem {
    // Contains only static inner System classes
}
```

## Architecture & Concepts

The WeatherSystem is not a traditional object but a logical container for a group of related systems within the Entity Component System (ECS) framework. It orchestrates the entire weather simulation on the server by separating responsibilities into distinct, event-driven processes. Its primary function is to manage the global weather state and synchronize it with individual players based on their location and other factors.

The module operates on two key data structures:
*   **WeatherResource:** A singleton resource within the ECS Store that holds the current global weather state for every environment type. It also manages forced weather states and update timers.
*   **WeatherTracker:** A component attached to player entities that tracks the last known weather state sent to that specific client. This prevents redundant network updates and enables smooth transitions.

The system is composed of four distinct inner classes, each handling a specific part of the weather lifecycle:

*   **WorldAddedSystem:** An initialization system that runs once when the world is created. It reads the world's configuration to apply any server-wide forced weather settings.
*   **PlayerAddedSystem:** A lifecycle system that automatically attaches a WeatherTracker component to new player entities, ensuring they are included in the weather simulation. It also cleans up the component when a player leaves.
*   **InvalidateWeatherAfterTeleport:** A reactive system that listens for Teleport events on entities. When a player teleports, it invalidates their local WeatherTracker, forcing a complete weather resynchronization to reflect their new environment.
*   **TickingSystem:** The core of the module. This system performs two functions on each game tick:
    1.  **Global Update:** It periodically checks the world time. On the hour, it consults the Environment asset definitions to randomly select new weather patterns for each environment, updating the global WeatherResource.
    2.  **Per-Player Synchronization:** It iterates over all players, comparing the global weather state with the player's tracked state. If a change is detected, it commands an update to be sent to the client, calculating the appropriate transition time.

## Lifecycle & Ownership

WeatherSystem itself is a static container and is never instantiated. The lifecycle described here pertains to the inner system classes (`TickingSystem`, `PlayerAddedSystem`, etc.).

*   **Creation:** The individual systems are discovered and instantiated by the ECS SystemGraph when a world's ECS Store is initialized. They are not created manually by developers.
*   **Scope:** Each system instance exists for the entire lifetime of the world (the ECS Store) to which it is attached.
*   **Destruction:** The systems are destroyed and cleaned up by the ECS framework when the world is unloaded. The `onSystemRemovedFromStore` callback in `WorldAddedSystem` is an example of this cleanup process.

## Internal State & Concurrency

*   **State:** The WeatherSystem module is stateless. All state is managed externally within the ECS framework through the mutable, world-global **WeatherResource** and the mutable, per-entity **WeatherTracker** components. Configuration data is read from immutable Environment assets.

*   **Thread Safety:** This system is **not thread-safe** for parallel execution. The core `TickingSystem` explicitly returns false from its `isParallel` method, forcing the ECS scheduler to execute its per-entity logic serially. This is a critical design choice to prevent race conditions when updating the global `WeatherResource` and individual `WeatherTracker` components. Any modifications to weather-related components or resources from other systems must be done via a `CommandBuffer` to ensure proper synchronization with the game loop.

## API Surface

The public contract of this module is not a set of methods on the `WeatherSystem` class, but rather the systems themselves and the components they operate on. Developers interact with this module by manipulating the ECS state that these systems query.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldAddedSystem | StoreSystem | O(1) | Initializes the global `WeatherResource` from world configuration upon world load. |
| PlayerAddedSystem | HolderSystem | O(1) | Manages the lifecycle of the `WeatherTracker` component on player entities. |
| InvalidateWeatherAfterTeleport | RefChangeSystem | O(1) | Resets a player's `WeatherTracker` after they teleport, forcing a weather resync. |
| TickingSystem | EntityTickingSystem | O(N) | The core update loop. N is the number of players. Updates global weather and syncs to players. |

## Integration Patterns

### Standard Usage

Direct interaction is uncommon. The primary way to influence the system is by modifying the global `WeatherResource`. For example, a game logic system for a magic spell could force a thunderstorm.

**WARNING:** All modifications to ECS state from other systems must be queued through a `CommandBuffer` to maintain thread safety and deterministic execution.

```java
// Example: A separate system forcing a weather change
ResourceType<EntityStore, WeatherResource> resourceType = WeatherResource.getResourceType();
WeatherResource weatherResource = store.getResource(resourceType);

// This is the correct way to trigger a global weather change.
// The WeatherSystem.TickingSystem will detect this change and propagate it.
weatherResource.setForcedWeather("hytale:thunderstorm");
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** The systems are managed by the ECS framework. Never use `new WeatherSystem.TickingSystem()`.
*   **Manual Component Management:** Do not manually add or remove the `WeatherTracker` component from players. The `PlayerAddedSystem` is the sole owner of this component's lifecycle. Manually altering it can lead to desynchronization.
*   **State Bypassing:** Do not modify the internal maps of `WeatherResource` directly. Always use the provided methods like `setForcedWeather`. Bypassing these methods will fail to trigger the necessary update flags that the `TickingSystem` relies on.

## Data Pipeline

The flow of data through the weather module is primarily driven by game events and the main tick loop.

> **Flow 1: World Initialization**
> World Configuration File (`forcedWeather` setting) -> `WorldAddedSystem` -> **WeatherResource** (Global State)

> **Flow 2: Periodic Weather Update**
> `WorldTimeResource` (Hour Change) -> `TickingSystem` (Global Tick) -> Environment Assets -> Random Selection -> **WeatherResource** (Global State)

> **Flow 3: Player Synchronization**
> **WeatherResource** (Global State) -> `TickingSystem` (Per-Entity Tick) -> `WeatherTracker` (Per-Player State) -> `CommandBuffer` -> Network Packet -> Client

