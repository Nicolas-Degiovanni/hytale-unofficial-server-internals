---
description: Architectural reference for TimeResource
---

# TimeResource

**Package:** com.hypixel.hytale.server.core.modules.time
**Type:** Data Model

## Definition
```java
// Signature
public class TimeResource implements Resource<EntityStore> {
```

## Architecture & Concepts
The TimeResource is a fundamental data component that encapsulates the state of time for a specific game world instance. It is not a service or a manager; rather, it is a passive data structure that holds the canonical "current moment" for a simulation.

Its implementation of the **Resource** interface signifies that it is a component attached to a broader container, in this case an **EntityStore**. This design pattern allows various engine systems to associate arbitrary data, like time, with core game objects like worlds.

The presence of a static **CODEC** field is a critical architectural feature. It indicates that TimeResource is deeply integrated with the server's persistence and serialization layers. The engine uses this codec to save the world's time to disk and to restore it upon loading, ensuring temporal continuity across sessions.

This component is exclusively managed by the **TimeModule**, which acts as the authoritative service that reads and updates the TimeResource during the main server game loop.

## Lifecycle & Ownership
- **Creation:** A TimeResource instance is created and attached to an EntityStore when a new world is initialized by the server. Alternatively, it is instantiated by the persistence framework via its CODEC when a world is loaded from storage.
- **Scope:** The lifetime of a TimeResource is strictly bound to the lifetime of its parent EntityStore. It persists in memory as long as the world it represents is active.
- **Destruction:** The object is marked for garbage collection when its parent EntityStore is unloaded from memory, for example, when a server shuts down or a world is unloaded. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The internal state is **highly mutable**. The core fields, *now* and *timeDilationModifier*, are expected to be updated frequently, typically once per server tick, to advance the game simulation.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. All reads and writes must be performed on the main server thread to prevent data corruption and race conditions. Off-thread access is a critical stability risk.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getNow() | Instant | O(1) | Retrieves the current, precise instant of in-game time. |
| setNow(Instant now) | void | O(1) | **Warning:** Forcefully sets the in-game time. Should only be used for initialization or administrative commands. |
| add(Duration duration) | void | O(1) | The primary method for advancing game time. Atomically adds the given duration to the current time. |
| setTimeDilationModifier(float) | void | O(1) | Sets the rate at which time progresses. 1.0 is normal speed, 0.5 is half speed. |
| clone() | Resource | O(1) | Creates a new TimeResource instance with an identical time value. Used for state snapshotting. |

## Integration Patterns

### Standard Usage
The TimeResource should never be directly instantiated or managed by user code. It must be retrieved from the appropriate EntityStore and manipulated by a controlling system, such as the main server tick loop.

```java
// Example from within a server-side system
EntityStore worldStore = server.getWorld().getStore();
TimeResource time = worldStore.getResource(TimeResource.getResourceType());

// Calculate the duration of the last game tick
Duration tickDelta = calculateTickDelta(); 

// Apply the time dilation factor
long dilatedNanos = (long)(tickDelta.toNanos() * time.getTimeDilationModifier());
Duration finalDelta = Duration.ofNanos(dilatedNanos);

// Advance the world's time
time.add(finalDelta);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new TimeResource()`. This creates a disconnected time state that is not part of the world simulation. Always retrieve the canonical instance from the EntityStore.
- **Multi-threaded Modification:** Do not access or modify a TimeResource from an asynchronous task or a different thread. All interactions must be synchronized with the main server game loop.
- **Arbitrary Time Jumps:** Avoid using `setNow` for normal time progression. It can cause severe desynchronization in time-dependent game logic (e.g., physics, animations, scheduled events). The `add` method ensures monotonic advancement.

## Data Pipeline
TimeResource acts as a stateful container within the server's data flow, not as a processing stage. Its primary role is to be updated by the game loop and read by the persistence layer.

> Flow:
> Server Tick Loop -> **TimeModule** (Calculates tick duration) -> `TimeResource.add()` -> **TimeResource** (Internal state updated) -> World Save Trigger -> Persistence Engine (Reads state via `CODEC`) -> Disk Storage

