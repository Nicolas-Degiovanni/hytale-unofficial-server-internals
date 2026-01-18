---
description: Architectural reference for ShutdownEvent
---

# ShutdownEvent

**Package:** com.hypixel.hytale.server.core.event.events
**Type:** Transient Event Object

## Definition
```java
// Signature
public class ShutdownEvent implements IEvent<Void> {
```

## Architecture & Concepts
The ShutdownEvent is a stateless, immutable signal used to orchestrate an orderly server shutdown. It is not a service or manager, but rather a message broadcast on the server's central EventBus. Its publication marks the beginning of the server's termination sequence.

The primary architectural significance of this event lies in its static priority constants: DISCONNECT_PLAYERS, UNBIND_LISTENERS, and SHUTDOWN_WORLDS. These constants define distinct, ordered phases of the shutdown process. Systems that need to perform cleanup during shutdown, such as the WorldManager or NetworkManager, subscribe to the ShutdownEvent using one of these priorities. This ensures that critical operations occur in a deterministic sequence: players are disconnected, then worlds are saved and unloaded, and finally network listeners are unbound. This phased approach prevents race conditions and data corruption, such as attempting to save a world after its players have been forcefully removed without a proper disconnect sequence.

This event is a critical component of the server's lifecycle state machine, facilitating the transition from a *Running* state to a *Stopped* state.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a high-level server management component when a shutdown is triggered. This can originate from a console command, an RCON signal, or a fatal, unrecoverable server error.
- **Scope:** Extremely short-lived. An instance of ShutdownEvent exists only for the duration of its propagation through the EventBus. It is a fire-and-forget message.
- **Destruction:** Eligible for garbage collection immediately after the EventBus has finished dispatching it to all registered subscribers. It holds no resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This class is stateless and immutable. An instance of ShutdownEvent contains no fields and carries no data. Its sole purpose is to act as a typed signal.
- **Thread Safety:** Inherently thread-safe due to its immutability. The event object can be safely passed between threads. The EventBus implementation is responsible for ensuring that listener execution is handled in a thread-safe manner, but the event object itself poses no concurrency risks.

## API Surface
The public contract of this class is defined by its type and its priority constants, not by methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DISCONNECT_PLAYERS | static final short | O(1) | Event priority for listeners responsible for disconnecting all connected players gracefully. |
| UNBIND_LISTENERS | static final short | O(1) | Event priority for listeners that unbind network ports and release socket resources. This is one of the final phases. |
| SHUTDOWN_WORLDS | static final short | O(1) | Event priority for listeners that handle saving world data to disk and unloading all world instances from memory. |

## Integration Patterns

### Standard Usage
Developers typically do not post this event. Instead, they create systems that listen for it to perform orderly cleanup. The standard pattern is to create a method in a service class, annotate it with a subscription annotation, and specify one of the predefined priorities.

```java
// Example of a WorldService listening for the shutdown signal
public class WorldService {

    @Subscribe(priority = ShutdownEvent.SHUTDOWN_WORLDS)
    public void onServerShutdown(ShutdownEvent event) {
        // This code is guaranteed to run after players are disconnected
        // but before the server fully terminates.
        log.info("Saving all worlds before shutdown...");
        saveAllWorldsToDisk();
        unloadAllWorlds();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Custom Priorities:** Do not subscribe to this event with an arbitrary or custom priority value. The shutdown sequence is carefully ordered. Using a priority that falls between the predefined constants can lead to unpredictable behavior and data corruption. Always use the provided static constants.
- **Event Cancellation:** Do not attempt to cancel or consume this event. It is a critical system-wide broadcast. Preventing its propagation would leave the server in a partially shut down, unstable state.
- **Triggering Manually:** Do not post a ShutdownEvent directly to the EventBus from general game logic. Server shutdown should only be initiated through designated high-level APIs, such as Server.shutdown(), which perform other essential checks before posting the event.

## Data Pipeline
ShutdownEvent does not process a pipeline of data; it initiates a broadcast-style command flow to multiple, independent systems in a prioritized order.

> Flow:
> Shutdown Trigger (e.g., Console Command) → Server#shutdown() → EventBus#post(new ShutdownEvent()) → **[Broadcast to Listeners]**
>
> **[Broadcast to Listeners]** → NetworkManager (Priority: DISCONNECT_PLAYERS)
>
> **[Broadcast to Listeners]** → WorldManager (Priority: SHUTDOWN_WORLDS)
>
> **[Broadcast to Listeners]** → NetworkListenerService (Priority: UNBIND_LISTENERS)

