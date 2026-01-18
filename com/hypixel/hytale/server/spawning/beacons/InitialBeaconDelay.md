---
description: Architectural reference for InitialBeaconDelay
---

# InitialBeaconDelay

**Package:** com.hypixel.hytale.server.spawning.beacons
**Type:** Data Component

## Definition
```java
// Signature
public class InitialBeaconDelay implements Component<EntityStore> {
```

## Architecture & Concepts
The InitialBeaconDelay is a stateful, short-lived data component within the server-side Entity Component System (ECS). Its sole purpose is to enforce a one-time, randomized delay on spawning logic associated with a "beacon" entity.

This component acts as a simple timer. It does not contain any logic for spawning entities itself. Instead, it is attached to an entity (the beacon) to signal to a governing system, such as a SpawningSystem, that this entity is in a waiting state. By assigning a random delay upon creation, this component effectively staggers the initial spawn waves from multiple beacons when a world chunk is first loaded. This prevents performance spikes caused by dozens of spawn points activating simultaneously.

Once the internal timer elapses, the component's responsibility is complete. The managing system is expected to perform the spawn action and then immediately remove this component from the entity.

## Lifecycle & Ownership
- **Creation:** An instance is typically added to an EntityStore by a higher-level system during the instantiation of a spawn beacon. The `setupInitialSpawnDelay` method is called immediately after creation to initialize the timer from a configuration range. The presence of a `clone` method suggests it may also be created from a pre-configured entity prototype.

- **Scope:** The component's lifetime is intentionally brief. It exists on an entity only from the moment the entity is loaded until its internal timer reaches zero. This typically spans a few seconds of server time.

- **Destruction:** The component is designed to be removed by its managing system as soon as `tickLoadTimeSpawnDelay` returns true. It is also implicitly destroyed if its parent EntityStore is unloaded or destroyed.

## Internal State & Concurrency
- **State:** The component's state is mutable, consisting of a single double field, `loadTimeSpawnDelay`. This value is initialized once and then decremented on every server tick until it is depleted.

- **Thread Safety:** This component is **not thread-safe**. Like most ECS components, it is designed to be accessed and mutated exclusively by a system running within the main server's single-threaded update loop. Unsynchronized access from other threads will lead to race conditions and unpredictable timer behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the SpawningPlugin. |
| setupInitialSpawnDelay(double[]) | void | O(1) | Initializes the internal timer with a random value within the provided min/max range. |
| tickLoadTimeSpawnDelay(float) | boolean | O(1) | Decrements the timer by the delta time. Returns true if the timer has elapsed, false otherwise. |
| clone() | Component | O(1) | Creates a new instance of the component with an identical timer value. |

## Integration Patterns

### Standard Usage
This component is not meant to be used directly by most game logic. It is exclusively managed by a dedicated spawning system that queries for entities possessing this component and ticks them once per server update.

```java
// Example from within a hypothetical SpawningSystem update method
for (EntityStore entity : world.query(InitialBeaconDelay.class)) {
    InitialBeaconDelay delayComponent = entity.getComponent(InitialBeaconDelay.class);

    // Tick the timer and check if it has finished
    if (delayComponent.tickLoadTimeSpawnDelay(deltaTime)) {
        // Timer is complete, proceed with spawning logic
        this.triggerSpawnFromBeacon(entity);

        // CRITICAL: Remove the component to prevent it from ever running again
        entity.removeComponent(InitialBeaconDelay.class);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not attempt to "reset" this component by calling `setupInitialSpawnDelay` after it has been used. The component is designed for a single countdown. Once its timer expires, it must be removed from the entity.

- **Manual Instantiation:** Do not instantiate this component without immediately attaching it to an EntityStore. A free-floating instance serves no purpose as no system will be aware of it.

- **External Ticking:** Do not call `tickLoadTimeSpawnDelay` from outside the primary server tick loop. The logic is dependent on a consistent delta time (`dt`) value to function correctly. Calling it manually or on other threads will break the timer.

## Data Pipeline
This component does not process a data stream. Instead, it represents a state within a control flow for staggering server load.

> Flow:
> Chunk Load -> Beacon Entity Instantiated -> **InitialBeaconDelay** component added -> SpawningSystem queries and finds component -> SpawningSystem calls `tickLoadTimeSpawnDelay` each frame -> Timer expires, method returns true -> SpawningSystem triggers spawn logic -> **InitialBeaconDelay** component is removed.

