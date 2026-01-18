---
description: Architectural reference for LocalSpawnController
---

# LocalSpawnController

**Package:** com.hypixel.hytale.server.spawning.local
**Type:** Transient Component Data

## Definition
```java
// Signature
public class LocalSpawnController implements Component<EntityStore> {
```

## Architecture & Concepts
The LocalSpawnController is not a controller in the traditional sense; it contains no business logic. It is a data-only **Component** within the server's Entity Component System (ECS) architecture. Its sole purpose is to act as a stateful timer, managing the cooldown period between local spawning attempts for the Entity to which it is attached.

This component is designed to be queried and manipulated by a dedicated spawning system, such as a hypothetical LocalSpawningSystem. The system is responsible for ticking the timer down and, upon expiration, executing the actual mob spawning logic. By decoupling the timing state (this component) from the spawning logic (the system), the engine allows for flexible and performant management of thousands of potential spawn locations.

The generic parameter `Component<EntityStore>` indicates that this component is attached to an Entity that exists within an EntityStore, the primary container for all server-side entities in a world.

## Lifecycle & Ownership
- **Creation:** A LocalSpawnController instance is typically created and attached to an Entity by a higher-level system. This can occur when a world chunk is loaded, a player enters a new area, or a spawner-type Entity is initialized. The initial timer value is retrieved from the central SpawningPlugin configuration.

- **Scope:** The lifecycle of a LocalSpawnController is strictly bound to the lifecycle of its parent Entity. It persists as long as the Entity exists in the EntityStore.

- **Destruction:** The component is destroyed and garbage collected when its parent Entity is removed from the world. There is no manual destruction method; cleanup is managed by the ECS framework.

## Internal State & Concurrency
- **State:** The component holds a single piece of mutable state: `timeToNextRunSeconds`. This double value acts as a countdown timer. It is initialized with a delay value and decremented by the game loop's delta time on each tick.

- **Thread Safety:** **This component is not thread-safe.** It is designed to be accessed and modified exclusively by the main server thread as part of the game's tick loop. Unsynchronized access from other threads will lead to race conditions and unpredictable timing behavior. All interactions must be marshaled through the main game loop's system updates.

## API Surface
The public contract is minimal, focusing entirely on managing the internal timer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setTimeToNextRunSeconds(double) | void | O(1) | Resets the internal countdown timer to the specified value in seconds. |
| tickTimeToNextRunSeconds(float) | boolean | O(1) | Decrements the timer by the delta time (dt). Returns true if the timer has reached or passed zero. |
| getComponentType() | ComponentType | O(1) | Static method to retrieve the unique type identifier for this component, used for efficient ECS queries. |

## Integration Patterns

### Standard Usage
The component is intended to be managed by a System. The system queries for entities with this component, ticks the timer, and acts when the timer expires.

```java
// Hypothetical usage within a LocalSpawningSystem's update method
public void update(float deltaTime) {
    EntityStore store = getUniverse().getWorld().getEntityStore();
    ComponentType<EntityStore, LocalSpawnController> type = LocalSpawnController.getComponentType();

    for (Entity entity : store.getEntitiesWithComponent(type)) {
        LocalSpawnController controller = entity.getComponent(type);

        // Tick the timer and check if it has expired
        if (controller.tickTimeToNextRunSeconds(deltaTime)) {
            // Timer expired, execute spawning logic
            executeSpawningLogicFor(entity);

            // Reset the timer for the next cycle
            double nextDelay = SpawningPlugin.get().calculateNextSpawnDelay();
            controller.setTimeToNextRunSeconds(nextDelay);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new LocalSpawnController()`. Components should be added to entities via the Entity or EntityStore APIs, which manage their lifecycle correctly.

- **External State Management:** Do not create a separate map or list to track spawn timers. The component is designed to live with its Entity, ensuring data locality and automatic cleanup.

- **Multi-threaded Modification:** Never call `setTimeToNextRunSeconds` or `tickTimeToNextRunSeconds` from an asynchronous task or worker thread without proper synchronization with the main server thread. This will corrupt the spawning schedule.

## Data Pipeline
This component is not part of a data processing pipeline. Instead, it serves as a state gate within the server's main game loop.

> Flow:
> Server Tick -> **LocalSpawningSystem** -> Reads/Updates **LocalSpawnController** state -> (If timer expired) -> Spawning Logic -> New Entity created in EntityStore

