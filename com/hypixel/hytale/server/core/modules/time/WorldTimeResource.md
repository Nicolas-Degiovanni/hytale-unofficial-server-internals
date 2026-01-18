---
description: Architectural reference for WorldTimeResource
---

# WorldTimeResource

**Package:** com.hypixel.hytale.server.core.modules.time
**Type:** Singleton (Per-World)

## Definition
```java
// Signature
public class WorldTimeResource implements Resource<EntityStore> {
```

## Architecture & Concepts
The WorldTimeResource is the canonical authority for the state and progression of time within a single game world. It is not a global singleton; each World instance possesses its own distinct WorldTimeResource, managed as a Resource within the world's primary EntityStore. This design ensures that different worlds (e.g., main world, dungeons, minigames) can maintain independent timelines.

This component's core responsibilities include:
- **Time Progression:** Advancing the in-game clock each server tick based on configurable day and night cycle durations.
- **State Calculation:** Deriving and caching critical time-dependent values such as the current sun position, sunlight intensity, moon phase, and a non-linear "scaled time" used for gameplay events.
- **Client Synchronization:** Serializing its state into network packets (UpdateTime, UpdateTimeSettings) and broadcasting them to all players within the world to ensure a consistent time-of-day experience.
- **API for Gameplay Systems:** Exposing a stable interface for other server systems (e.g., mob spawning, crop growth, NPC schedules) to query the current time, sun/moon state, and other temporal conditions.

The class encapsulates complex temporal mechanics, translating a simple, linear progression of server ticks into a nuanced and configurable in-game day-night cycle. It directly reads configuration from the World and WorldConfig objects to determine cycle lengths and whether time is paused.

### Lifecycle & Ownership
- **Creation:** A WorldTimeResource is instantiated and attached to an EntityStore when a new World is created and initialized by the server. It is not intended for manual creation.
- **Scope:** The resource's lifetime is strictly bound to its parent World. It persists as long as the world is loaded in memory.
- **Destruction:** The resource is dereferenced and becomes eligible for garbage collection when its parent World is unloaded from the server.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. It maintains the canonical game time as a Java Instant, along with numerous cached, derived values like `_gameTimeLocalDateTime`, `sunlightFactor`, `scaledTime`, and `moonPhase`. It also holds pre-allocated network packet objects (`currentTimePacket`, `currentSettings`) to reduce allocation overhead during the game loop.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be exclusively owned and mutated by the single main thread that ticks the associated World. All state-mutating methods, especially `tick` and `setGameTime`, must only be called from this thread. Unsynchronized access from other threads (e.g., for console commands) will lead to state corruption, inconsistent client updates, and severe gameplay bugs. Any cross-thread interaction must be marshaled through a thread-safe queue to the main world thread.

## API Surface
The public API provides methods for advancing, setting, and querying the world's time.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, store) | void | O(N) | Advances time by delta `dt`. Complexity is O(N) where N is the number of players, due to network broadcasts. This is the primary update method. |
| setGameTime(time, world, store) | void | O(N) | Forcefully sets the world time to a specific Instant. Broadcasts the change to all players. |
| setDayTime(dayTime, world, store) | void | O(N) | Sets the time to a specific point in the 24-hour cycle, represented as a double between 0.0 and 1.0. |
| getGameTime() | Instant | O(1) | Returns the precise, canonical game time. |
| getSunlightFactor() | double | O(1) | Returns the calculated sunlight intensity, from 0.0 (night) to 1.0 (midday). |
| getSunDirection() | Vector3f | O(1) | Calculates and returns the normalized vector pointing away from the sun, used for lighting and shadow rendering. |
| getMoonPhase() | int | O(1) | Returns the current moon phase index. |
| sendTimePackets(playerRef) | void | O(1) | Sends the current time and settings packets to a single, specific player. Used for players just joining the world. |

## Integration Patterns

### Standard Usage
The WorldTimeResource is driven by the server's main game loop. Other game systems should retrieve the resource from the world's EntityStore to query the time state for their logic.

```java
// Within a system that has access to the world's store
// This is the correct way to get the time resource for a given world.
WorldTimeResource time = store.getResource(WorldTimeResource.getResourceType());

// Example: A mob spawning system checking if it's night
if (time.getSunlightFactor() < 0.1) {
    // ... spawn hostile mobs
}

// Example: An admin command setting the time to noon
double noon = 0.5;
time.setDayTime(noon, world, store);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new WorldTimeResource()`. The resource is managed by the world's lifecycle. Direct instantiation will result in a disconnected time state that is not ticked or synchronized with clients.
- **State Inconsistency:** Modifying the `Instant` object returned by `getGameTime()` will have no effect and can lead to confusion. To change the time, you must use a designated mutator method like `setGameTime` or `setDayTime`, which ensures all derived data is recalculated and changes are broadcast to clients.
- **Asynchronous Modification:** Calling `setGameTime` or any other mutator from a separate thread without proper synchronization is strictly forbidden. This will cause race conditions with the main `tick` method, corrupting the time state.

## Data Pipeline
The WorldTimeResource primarily acts as a data source and state machine, driven by the server tick.

> **Time Progression Flow:**
> Server Game Loop -> `World.tick()` -> **WorldTimeResource.tick(dt)** -> Internal `gameTime` is advanced -> Derived state (sunlight, etc.) is recalculated -> `UpdateTime` packet is populated -> `PlayerUtil.broadcastPacketToPlayers` -> Network Layer -> All World Clients

> **State Query Flow:**
> Gameplay System (e.g., Spawner) -> `store.getResource(...)` -> **WorldTimeResource.getSunlightFactor()** -> Return cached sunlight value -> Gameplay Logic Execution

