---
description: Architectural reference for BootEvent
---

# BootEvent

**Package:** com.hypixel.hytale.server.core.event.events
**Type:** Transient

## Definition
```java
// Signature
public class BootEvent implements IEvent<Void> {
```

## Architecture & Concepts
The BootEvent is a foundational *marker event* within the server's startup sequence. It carries no data payload, as indicated by its implementation of IEvent<Void>. Its sole purpose is to signal that the core server systems have completed their initial bootstrap phase and are ready to proceed to the next stage of operation.

This event acts as a critical synchronization point. It decouples the main server initialization thread from the various subsystems that must perform post-boot actions. By listening for this event, components such as the PluginManager, WorldLoader, or NetworkAcceptor can reliably begin their work without needing direct, hard-coded dependencies on the bootstrap controller. This promotes a clean, observable, and extensible startup architecture.

## Lifecycle & Ownership
- **Creation:** A single instance of BootEvent is created by the primary server entry point (e.g., ServerMain or a similar bootstrap class) precisely once per server session. This occurs after all essential services have been instantiated and registered but before the server enters its main game loop or begins accepting player connections.

- **Scope:** The object's lifetime is exceptionally brief. It exists only for the duration of its dispatch through the central EventManager.

- **Destruction:** Once the EventManager has delivered the event to all registered listeners, the BootEvent instance is no longer referenced and becomes immediately eligible for garbage collection.

## Internal State & Concurrency
- **State:** **Immutable**. The BootEvent class contains no fields and therefore has no internal state. It is a pure signaling mechanism.

- **Thread Safety:** **Inherently Thread-Safe**. As a stateless, immutable object, a BootEvent instance can be safely published and consumed across any thread without locks or other synchronization primitives. The responsibility for thread-safe event *handling* resides entirely within the listener implementations.

## API Surface
The public contract of BootEvent is its type, not its methods. It has no meaningful API beyond the standard Object methods. Its value is in being an instance of BootEvent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toString() | String | O(1) | Returns a static string representation of the event. |

## Integration Patterns

### Standard Usage
Systems do not create or interact with a BootEvent directly. Instead, they subscribe to it to trigger their own initialization logic. The standard pattern is to define a listener method within a service class that is registered with the server's EventManager.

```java
// Example: A PluginLoader service listening for the boot signal
public class PluginLoaderService {

    @Subscribe
    public void onServerBoot(BootEvent event) {
        // The server has finished its core bootstrap.
        // It is now safe to discover and load all plugins.
        this.loadPluginsFromDisk();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation and Firing:** Never create and post a BootEvent from application-level code (e.g., from within a game script or plugin). This is a system-level event with a specific, guaranteed timing. Firing it manually will cause severe state desynchronization, likely leading to race conditions and server crashes as systems initialize prematurely.

- **State Assumption:** Do not assume that this event signifies that the server is ready to accept players or that a world has been loaded. It only signals the completion of the *initial bootstrap*. Other events, such as WorldLoadEvent or ServerReadyEvent, will signal subsequent states.

## Data Pipeline
As a marker event, BootEvent does not carry data. Its pipeline represents a flow of control and signals a state transition within the server.

> Flow:
> Core Server Bootstrap -> **new BootEvent()** -> EventManager.post() -> Registered Listeners (e.g., PluginLoader, WorldManager) -> Further System Initialization

