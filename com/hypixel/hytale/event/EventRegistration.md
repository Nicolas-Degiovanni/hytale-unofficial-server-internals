---
description: Architectural reference for EventRegistration
---

# EventRegistration

**Package:** com.hypixel.hytale.event
**Type:** Transient

## Definition
```java
// Signature
public class EventRegistration<KeyType, EventType extends IBaseEvent<KeyType>> extends Registration {
```

## Architecture & Concepts
The EventRegistration class is a foundational component of the engine's event handling system. It is not the event bus itself, but rather a *handle* or *receipt* that represents a single, specific subscription of a listener to an event type. Its primary purpose is to provide a lifecycle-managed reference to an active event listener, enabling explicit control over its registration state.

This class extends the base Registration class, inheriting the core concepts of an enabled/disabled state and an unregistration mechanism. The key architectural features are:

*   **Explicit Lifecycle:** By returning an EventRegistration object upon subscription, the event system forces the caller to manage the listener's lifecycle. The owner of the listener is responsible for holding this handle and calling its unregister method to prevent memory and performance leaks.
*   **Conditional Listening:** The `isEnabled` supplier allows for dynamic and declarative control over a listener's activity. A listener can be temporarily disabled without the cost of unregistering and re-registering, for example, when a UI screen is hidden but not destroyed.
*   **Compositional Design:** The static `combine` method is a powerful pattern that allows multiple EventRegistration objects to be composed into a single, logical registration. The resulting combined registration is only considered enabled if *all* of its constituent registrations are enabled. This facilitates building robust, hierarchical systems where the validity of a child component's listener is dependent on the state of its parent or container.

## Lifecycle & Ownership
-   **Creation:** EventRegistration instances are created and returned by a central EventManager or EventBus service when a system registers a new event listener. They are also created internally by the static `combine` factory method. Direct instantiation by client code is a severe anti-pattern.
-   **Scope:** The object's lifetime is directly tied to the intended lifetime of the event subscription it represents. The system that creates the subscription (e.g., a UI widget, a game entity) is responsible for storing the EventRegistration handle in a field. It persists as long as its owner requires the subscription to be potentially active.
-   **Destruction:** The underlying event subscription is terminated when the `unregister` method (inherited from Registration) is invoked. This is a manual process that must be performed by the owner of the handle, typically during its own teardown phase (e.g., in a `dispose` or `onRemoved` method). Failure to unregister will result in a permanent listener, causing memory leaks and unintended behavior.

## Internal State & Concurrency
-   **State:** An EventRegistration object is effectively immutable after construction. It holds final references to the event class and the functional interfaces (`BooleanSupplier`, `Runnable`) provided at creation. It is a lightweight data carrier whose primary state is the encapsulated behavior of its functional members.
-   **Thread Safety:** The class itself contains no internal locks or synchronization and is safe to be passed between threads. However, its thread safety is entirely dependent on the thread safety of the `BooleanSupplier` and `Runnable` implementations provided during its construction.

    **WARNING:** If the `isEnabled` supplier or `unregister` runnable access mutable state from multiple threads, it is the responsibility of the implementer to ensure proper synchronization. The EventManager may check the `isEnabled` status from any thread where events are posted.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEventClass() | Class<EventType> | O(1) | Returns the specific event class this registration is bound to. |
| combine(...) | EventRegistration | O(N) | A static factory that composes multiple registrations into a new, single registration. The resulting handle is enabled only if all constituents are enabled. |

## Integration Patterns

### Standard Usage
The typical pattern involves registering a listener, storing the returned handle, and using it for cleanup. The `combine` method is used for managing dependent listeners.

```java
// Assumes an EventManager service exists in the context
class PlayerUISystem {
    private EventRegistration playerHealthRegistration;
    private EventRegistration uiVisibleRegistration;

    public void initialize(Context context, UIContainer parent) {
        EventManager eventManager = context.getService(EventManager.class);

        // 1. Register a listener for a specific event
        EventRegistration healthListener = eventManager.register(PlayerHealthChangedEvent.class, this::onHealthChanged);

        // 2. Create a registration representing the UI's lifecycle
        EventRegistration uiLifecycle = parent.getLifecycleRegistration();

        // 3. Combine them: the health listener is only active if the UI is also active
        this.playerHealthRegistration = EventRegistration.combine(healthListener, uiLifecycle);
    }

    public void shutdown() {
        // 4. Unregister the listener during cleanup to prevent leaks
        if (this.playerHealthRegistration != null) {
            this.playerHealthRegistration.unregister();
        }
    }

    private void onHealthChanged(PlayerHealthChangedEvent event) {
        // ... update UI
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new EventRegistration()`. This creates a handle that is not tracked by the engine's EventManager, and the `unregister` runnable will not be correctly wired to the central bus. Always acquire instances from the appropriate registration service.
-   **Discarding the Handle:** Registering a listener and immediately discarding the returned EventRegistration is a guaranteed memory leak. The listener will remain active for the entire application session with no possibility of removal.
-   **Unsafe Lambdas:** Providing a non-thread-safe `BooleanSupplier` to a registration can lead to race conditions if the event bus is multi-threaded, potentially causing crashes or inconsistent listener behavior.

## Data Pipeline
EventRegistration does not process data itself. It acts as a control gate within the larger event data pipeline, enabling or disabling a listener's participation.

> Flow:
> 1. **Registration Time:**
>    Listener Code -> `EventManager.register(listener)` -> **EventRegistration (Handle)** -> Stored by Listener's Owner
>
> 2. **Event Dispatch Time:**
>    Event Source -> `EventManager.post(event)` -> EventManager iterates listeners -> Checks **EventRegistration.isEnabled()** -> If true, invokes listener
>
> 3. **Teardown Time:**
>    Owner Component -> **EventRegistration.unregister()** -> Listener is removed from EventManager's dispatch list

