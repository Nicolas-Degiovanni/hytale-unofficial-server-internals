---
description: Architectural reference for LoadAssetEvent
---

# LoadAssetEvent

**Package:** com.hypixel.hytale.server.core.asset
**Type:** Transient

## Definition
```java
// Signature
public class LoadAssetEvent implements IEvent<Void> {
```

## Architecture & Concepts
The LoadAssetEvent is a foundational control-flow object within the server's startup sequence. It is not a service or manager, but rather a specialized **Message** used within the engine's event-driven architecture. Its primary purpose is to orchestrate the multi-stage process of loading game assets and to aggregate the results of these operations.

This event is dispatched a single time by the server's main entry point during bootstrap. Multiple systems, such as registry loaders and resource managers, subscribe to this event. The defined priority constants (**PRIORITY_LOAD_COMMON**, **PRIORITY_LOAD_REGISTRY**, **PRIORITY_LOAD_LATE**) enforce a strict, sequential loading order, ensuring that foundational assets are available before dependent systems attempt to load.

During its lifecycle, the event object acts as a mutable, shared state container. Listeners report failures back to the event instance, which accumulates error reasons and determines if a failure is catastrophic enough to warrant a server shutdown.

### Lifecycle & Ownership
-   **Creation:** Instantiated once by the primary server bootstrap coordinator (e.g., ServerEntryPoint) at the beginning of the server startup process. The initial boot timestamp is captured upon creation for performance monitoring.
-   **Scope:** The object's lifetime is strictly limited to the initial asset loading phase. It is a short-lived object that does not persist after the server has successfully started.
-   **Destruction:** After the event bus has finished dispatching the event to all subscribed listeners, the bootstrap coordinator inspects its final state. Following this inspection, all references to the object are dropped, and it becomes eligible for garbage collection.

## Internal State & Concurrency
-   **State:** The LoadAssetEvent is inherently **mutable**. Its design requires listeners to modify its internal state by adding failure reasons and updating the shutdown flag. The internal list of reasons is backed by fastutil's ObjectArrayList for performance.
-   **Thread Safety:** This class is **not thread-safe** and provides no internal locking. It is architecturally mandated that the event bus dispatches this specific event sequentially on the main server thread. Any attempt to modify this event from an asynchronous worker thread without external synchronization will result in a race condition and undefined behavior.

## API Surface
The public API is designed for two distinct consumers: the event listeners who report status, and the bootstrap coordinator who inspects the final result.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| failed(shouldShutdown, reason) | void | O(1) Amortized | The primary mutation method for listeners. Adds a failure reason and logically ORs the shutdown flag. Once `shouldShutdown` is true, it cannot be reverted to false. |
| isShouldShutdown() | boolean | O(1) | Returns true if any listener has reported a failure that necessitates a server shutdown. |
| getReasons() | List<String> | O(1) | Returns a reference to the internal list of all reported failure reasons. **Warning:** The returned list is mutable. |
| getBootStart() | long | O(1) | Returns the system timestamp captured when the server boot process began. |

## Integration Patterns

### Standard Usage
A system that needs to load assets during startup will subscribe to this event. The bootstrap coordinator is responsible for dispatching the event and acting on its result.

```java
// In a listener class (e.g., RegistryLoader)
@Subscribe(priority = LoadAssetEvent.PRIORITY_LOAD_REGISTRY)
public void onAssetLoad(LoadAssetEvent event) {
    try {
        // ... perform complex asset loading ...
    } catch (Exception e) {
        // Report a critical failure
        event.failed(true, "Failed to load critical block registry.");
    }
}

// In the main server bootstrap logic
LoadAssetEvent event = new LoadAssetEvent(System.currentTimeMillis());
eventBus.post(event);

if (event.isShouldShutdown()) {
    // Log event.getReasons() and terminate the server
    System.exit(1);
}
```

### Anti-Patterns (Do NOT do this)
-   **Listener Instantiation:** An event listener must **never** create its own instance of LoadAssetEvent. It must only operate on the instance provided by the event bus.
-   **Asynchronous Modification:** Do not perform asset loading on a separate thread and attempt to call `failed` from that thread. The event's state is not protected against concurrent access.
-   **State Clearing:** Do not attempt to call `getReasons().clear()` or otherwise reset the state of the event. It is designed as a write-once, aggregate-many message for a single, atomic operation.

## Data Pipeline
The LoadAssetEvent does not process a stream of data; instead, it represents the flow of control and state during the server startup phase.

> Flow:
> Server Entry Point -> Instantiates **LoadAssetEvent** -> Event Bus Dispatch -> Listener (Common) -> Listener (Registry) -> ... -> Listener (Late) -> Server Entry Point (Inspects final state) -> Continue Boot or Initiate Shutdown

