---
description: Architectural reference for StartWorldEvent
---

# StartWorldEvent

**Package:** com.hypixel.hytale.server.core.universe.world.events
**Type:** Transient

## Definition
```java
// Signature
public class StartWorldEvent extends WorldEvent {
```

## Architecture & Concepts
The StartWorldEvent is a fundamental signaling mechanism within the server's event-driven architecture. It represents a critical, non-revertible state transition in the lifecycle of a World instance. Its primary role is to decouple the world management system from the numerous subsystems that must react to a world becoming active.

This class is not a service or a manager; it is a pure data object, an immutable message. When the core server logic decides a World is ready for simulation, it instantiates and dispatches a StartWorldEvent onto the main server event bus. Subsystems such as the AI scheduler, player spawn logic, and scripting engines subscribe to this specific event type. Upon receipt, they perform their world-specific initialization tasks.

The existence of this event signifies that the World's data structures are loaded and initialized, but the primary simulation loop (ticking) has not yet begun. Listeners for this event are expected to perform the final setup before the first tick.

## Lifecycle & Ownership
- **Creation:** A StartWorldEvent is instantiated exclusively by the high-level world management service (e.g., a WorldManager or UniverseManager) at the exact moment a World transitions from a loading or dormant state to an active one.
- **Scope:** This object is extremely short-lived. Its scope is confined to the duration of its dispatch on the event bus. Once all registered listeners have processed the event, it becomes eligible for garbage collection.
- **Destruction:** The Java Garbage Collector reclaims the memory for the event object. Listeners **must not** retain a reference to the event object itself beyond the scope of their handler method.

## Internal State & Concurrency
- **State:** The StartWorldEvent is stateless beyond the immutable reference to the World it encapsulates. This reference is inherited from its parent, WorldEvent, and is set once during construction.
- **Thread Safety:** This event object is inherently thread-safe due to its immutability. It can be safely passed across thread boundaries. However, the contained World object is **not** guaranteed to be thread-safe. All interactions with the World instance obtained from this event must adhere to the server's main thread synchronization model. Any listener operating on a separate thread must dispatch its work back to the main world thread.

## API Surface
The public contract is minimal, consisting of the inherited accessor from its parent class. The constructor is not intended for public use outside the core world management system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWorld() | World | O(1) | Retrieves the non-null World instance that is beginning its simulation. |

## Integration Patterns

### Standard Usage
The canonical use case is to implement a listener that subscribes to the event on the server's event bus. The handler method performs initialization logic scoped to the specific world.

```java
// Example of a subsystem listening for the event
public class ScriptingEngine {

    @Subscribe
    public void onWorldStart(StartWorldEvent event) {
        World world = event.getWorld();
        // The world is ready, but not yet ticking.
        // This is the correct place to load and initialize all world-specific scripts.
        loadScriptsForWorld(world.getId());
        LOGGER.info("Scripts loaded for world: " + world.getName());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of StartWorldEvent in subsystem code. Firing a rogue event would cause other systems to perform duplicate, and likely catastrophic, initializations on an already running world. Event creation is the sole responsibility of the authoritative WorldManager.
- **Delayed Processing:** Do not queue this event for later processing. The event signifies a precise moment in the server lifecycle. Logic that depends on it must run synchronously within the event handler to guarantee correct system ordering.

## Data Pipeline
The flow of this event is linear and synchronous, acting as a barrier before the world simulation begins.

> Flow:
> WorldManager -> **new StartWorldEvent(world)** -> Server EventBus -> (Broadcast to all listeners) -> AI System, Scripting Engine, Player Manager -> World is marked as fully running -> Simulation Loop Begins

