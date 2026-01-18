---
description: Architectural reference for TimeSystem
---

# TimeSystem

**Package:** com.hypixel.hytale.server.core.modules.time
**Type:** System

## Definition
```java
// Signature
public class TimeSystem extends TickingSystem<EntityStore> {
```

## Architecture & Concepts
The TimeSystem is a fundamental component of the server-side game loop, acting as the primary driver for the in-game world clock. As a subclass of TickingSystem, it is designed to be executed once per simulation tick by a higher-level system scheduler.

Its sole responsibility is to advance a global TimeResource based on the delta time provided by the core engine. It does not manage time itself; rather, it translates the engine's tick duration into a high-precision nanosecond value and applies it to a shared resource within the world's data Store. This decouples the concept of world time from the system that updates it, allowing other game systems to query the current time from the central Store without needing a direct reference to the TimeSystem.

This class embodies the "S" (Systems) in an Entity-Component-System (ECS) architecture, operating on global resources rather than individual entities.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's world loader or system manager during the world initialization phase. Its dependencies, specifically the ResourceType for the TimeResource, are injected at this time. It is not intended for manual creation.
- **Scope:** The lifecycle of a TimeSystem instance is bound to the lifecycle of the server world it manages. It persists as long as the world is loaded and running.
- **Destruction:** The instance is marked for garbage collection when its parent world is unloaded or the server shuts down. The owning system manager is responsible for de-registering it from the tick loop.

## Internal State & Concurrency
- **State:** The TimeSystem is effectively stateless. Its only instance field, timeResourceType, is a final reference injected during construction. It does not cache or store any data between ticks. All state modifications are performed on the external TimeResource object located within the shared Store.
- **Thread Safety:** This class is **not thread-safe** and must only be operated on by the main server thread. The TickingSystem contract assumes a single-threaded, sequential execution model for the game loop. Concurrent calls to the tick method will result in race conditions and corrupt the global TimeResource.

## API Surface
The public API is exclusively for the engine's system scheduler. Game logic developers should not invoke these methods directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | O(1) | Advances the world's TimeResource by the delta time. Invoked once per game tick by the system scheduler. |

## Integration Patterns

### Standard Usage
Developers should not interact with the TimeSystem directly. Instead, they should query the TimeResource from the world's Store to get the current time for game logic calculations.

```java
// Correctly accessing the data produced by TimeSystem
// (Example assumes access to the world's Store and the TimeResource type)

TimeResource time = store.getResource(TIME_RESOURCE_TYPE);
long currentTimeNanos = time.getNanos();
// Use currentTimeNanos for gameplay logic...
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new TimeSystem()`. The system must be created and registered by the world's system manager to be included in the tick cycle. Manual instantiation creates an orphaned object that does nothing.
- **Manual Ticking:** Do not call the `tick` method from your own code. This will advance the world clock out of sync with all other game simulation systems, causing severe logic errors and desynchronization.

## Data Pipeline
The TimeSystem acts as a simple processor in the main game loop's data flow, converting a temporal engine value into a persistent world state.

> Flow:
> Game Loop Scheduler -> Provides `dt` (float) -> **TimeSystem.tick()** -> Updates `TimeResource` (long) -> In World `Store`

