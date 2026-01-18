---
description: Architectural reference for IAsyncEvent
---

# IAsyncEvent

**Package:** com.hypixel.hytale.event
**Type:** Contract / Marker Interface

## Definition
```java
// Signature
public interface IAsyncEvent<KeyType> extends IBaseEvent<KeyType> {
    // This interface defines no methods of its own.
}
```

## Architecture & Concepts
The IAsyncEvent interface is a foundational component of Hytale's concurrency model. It serves as a **marker interface**, signaling to the core EventManager that any event class implementing it is designed for asynchronous, off-main-thread processing.

Its primary architectural purpose is to decouple long-running or blocking operations from the main game loop. By classifying an event as asynchronous, systems like networking, file I/O, or complex data processing can notify other parts of the engine of their completion without introducing latency or stuttering to the client's frame rate.

When the EventManager receives an object that implements IAsyncEvent, it does not dispatch it immediately on the calling thread. Instead, it enqueues the event to be processed by a dedicated worker thread pool. This ensures that the main thread remains unblocked and responsive, which is critical for maintaining a smooth user experience.

## Lifecycle & Ownership
As an interface, IAsyncEvent itself has no lifecycle. The following pertains to the concrete event objects that implement this interface.

- **Creation:** Event objects implementing IAsyncEvent are instantiated by various engine subsystems when a non-blocking operation completes or a background task needs to report its results. For example, a NetworkSystem might create a ProfileDataReceivedEvent after successfully fetching player data from a remote server.
- **Scope:** The scope of an asynchronous event object is extremely transient. It exists only for the duration of its dispatch cycle: from instantiation, through the EventManager's queue, to the completion of all its registered listeners on a worker thread.
- **Destruction:** Once all listeners have processed the event, the EventManager releases its reference. The event object is then immediately eligible for garbage collection. Holding references to event objects after they have been posted is a severe anti-pattern.

## Internal State & Concurrency
- **State:** The state of an IAsyncEvent is defined by its concrete implementation. The data contained within the event is inherently mutable upon creation but must be treated as **effectively immutable** after being posted to the EventManager. The posting thread cedes ownership of the object and must not modify it further to prevent severe data races.
- **Thread Safety:** This interface makes a strong assertion about thread safety. Any class implementing IAsyncEvent is, by contract, safe to be read from a different thread than the one it was created on. Furthermore, all listeners registered for an IAsyncEvent **must be thread-safe**, as they will be executed concurrently on a background worker thread. Listeners must not access or modify main-thread-only game state without proper synchronization mechanisms, such as queueing a subsequent synchronous event.

## API Surface
This interface adds no new methods. Its public contract is its type, which signals its asynchronous nature to the EventManager. It inherits the contract of IBaseEvent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| (Inherited from IBaseEvent) | - | - | Provides the base contract for all engine events. |

## Integration Patterns

### Standard Usage
The standard pattern is to define a concrete event class for a specific asynchronous operation, implement IAsyncEvent, and post it to the global EventManager.

```java
// 1. Define a concrete event for a specific background task
public class PlayerDataDownloadedEvent implements IAsyncEvent<Void> {
    private final PlayerData data;

    public PlayerDataDownloadedEvent(PlayerData data) {
        this.data = data;
    }

    public PlayerData getData() {
        return this.data;
    }
}

// 2. Post the event from a background service
// (Inside a network callback or worker thread)
PlayerData result = downloadData();
eventManager.post(new PlayerDataDownloadedEvent(result));
```

### Anti-Patterns (Do NOT do this)
- **Modifying Game World State:** Do not implement IAsyncEvent for any event that requires immediate and direct modification of critical game state (e.g., entity positions, physics volumes). Such operations must be synchronous to prevent race conditions and world state corruption.
- **Reusing Event Objects:** Event objects are single-use. Do not store and re-post an instance of an IAsyncEvent.
- **Assuming Main Thread Context:** Listeners for an IAsyncEvent must never assume they are running on the main game thread. Accessing non-thread-safe objects like UI components or the scene graph will lead to crashes or undefined behavior.

## Data Pipeline
The flow of an asynchronous event is fundamentally different from a synchronous one, involving a handoff between threads.

> Flow:
> Background Operation (e.g., File Read) -> Event Instantiation -> `EventManager.post()` -> **IAsyncEvent detected** -> Event placed in Worker Queue -> Worker Thread Pool picks up Event -> **Listeners Executed on Worker Thread**

