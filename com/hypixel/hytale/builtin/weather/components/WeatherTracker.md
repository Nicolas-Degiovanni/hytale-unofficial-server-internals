---
description: Architectural reference for WeatherTracker
---

# WeatherTracker

**Package:** com.hypixel.hytale.builtin.weather.components
**Type:** Transient Component

## Definition
```java
// Signature
public class WeatherTracker implements Component<EntityStore> {
```

## Architecture & Concepts
The WeatherTracker is a server-side, stateful component responsible for managing the weather conditions perceived by a single entity, typically a player. It acts as the bridge between the static environmental data of the world and the dynamic weather state sent to the client.

As part of Hytale's Entity Component System (ECS), an instance of WeatherTracker is attached to an entity to track its position. By monitoring the entity's `TransformComponent`, it detects when the entity moves into a new block position. Upon movement, it queries the world's `ChunkStore` to determine the environment ID of the new location. This ID is then used to look up the appropriate weather type from a `WeatherResource` configuration file.

Its primary function is to decide when to send an `UpdateWeather` network packet to the client, ensuring the player's view of the weather is synchronized with their location and any server-enforced weather events. It intelligently avoids sending redundant packets if the weather state has not changed.

## Lifecycle & Ownership
- **Creation:** A WeatherTracker component is added to an entity, typically a player, by a higher-level game system. This usually occurs when the entity is spawned into a world. The presence of a `clone` method indicates it is designed to be duplicated, a common pattern for entity templating and serialization within the ECS framework.
- **Scope:** The component's lifetime is strictly tied to the entity it is attached to. It persists as long as the entity exists within an `EntityStore`.
- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the world or destroyed. The `clear` method provides a mechanism to reset its state, which may be used during events like player respawn or world transfers without destroying the entity itself.

## Internal State & Concurrency
- **State:** This component is highly mutable and stateful. It maintains the entity's last known block position (`previousBlockPosition`), the current environment ID, and the last weather index sent to the client. It also caches an `UpdateWeather` packet object to reduce allocations. The `firstSendForWorld` flag manages initial state synchronization when an entity enters a world.
- **Thread Safety:** This component is **not thread-safe**. It is designed to be accessed and mutated by a single thread within the main server tick, managed by the ECS. Unsynchronized access from multiple threads will lead to race conditions, particularly when reading and writing `previousBlockPosition` and `updateWeather`. All interactions must be marshaled through the appropriate game system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| updateWeather(...) | void | O(C) | The primary update method. Checks for forced weather, updates the environment based on position, and sends new weather state to the client. Complexity depends on chunk lookup. |
| sendWeatherIndex(...) | void | O(1) | Sends an `UpdateWeather` packet to the associated player if the new weather index differs from the current one. |
| consumeFirstSendForWorld() | boolean | O(1) | A state-checking method to handle the initial weather packet sent when an entity joins a world. Returns true only on the first call after a reset. |
| clear() | void | O(1) | Resets the internal state, primarily for reuse or on player respawn. |
| updateEnvironment(...) | void | O(C) | Core logic for detecting positional changes and querying the world's chunk data to get the current environment ID. |

## Integration Patterns

### Standard Usage
The WeatherTracker is managed by a server-side system, such as a `WeatherSystem`. This system iterates over all entities possessing a `WeatherTracker`, `TransformComponent`, and `PlayerRef` on each game tick, calling `updateWeather` to process any changes.

```java
// Example within a hypothetical WeatherSystem
void onTick(EntityRef entity, ComponentAccessor<EntityStore> accessor) {
    WeatherTracker tracker = accessor.getComponent(entity, WeatherTracker.getComponentType());
    TransformComponent transform = accessor.getComponent(entity, TransformComponent.getComponentType());
    PlayerRef player = accessor.getComponent(entity, PlayerRef.getComponentType());
    WeatherResource weatherRules = world.getResource(WeatherResource.class);

    if (tracker != null && transform != null && player != null) {
        float transitionTime = 10.0f; // Default transition
        tracker.updateWeather(player, weatherRules, transform, transitionTime, accessor);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new WeatherTracker()`. Components must be created and managed by the `EntityStore` or `ComponentAccessor` to ensure they are correctly registered within the ECS.
- **External State Mutation:** Do not modify internal fields like `previousBlockPosition` or `environmentId` from outside the class. This will corrupt the component's state and lead to unpredictable behavior. Use the public API.
- **Ignoring Accessors:** Bypassing the `ComponentAccessor` to get a reference to the component can break thread-safety guarantees and lead to severe concurrency bugs. Always operate on components through the accessors provided by the game system.

## Data Pipeline
The component processes positional data to produce network packets. It acts as a stateful filter, translating an entity's continuous movement through the world into discrete weather state updates for the client.

> Flow:
> Entity Position (from `TransformComponent`) -> **WeatherTracker.updateEnvironment** -> World Chunk Lookup -> Environment ID -> `WeatherResource` Mapping -> Weather Index -> **WeatherTracker.sendWeatherIndex** -> `UpdateWeather` Network Packet -> Player Client

---

