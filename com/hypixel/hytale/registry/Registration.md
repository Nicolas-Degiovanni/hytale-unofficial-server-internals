---
description: Architectural reference for Registration
---

# Registration

**Package:** com.hypixel.hytale.registry
**Type:** Transient

## Definition
```java
// Signature
public class Registration {
```

## Architecture & Concepts
The Registration class is a stateful handle that represents a single, managed registration within a higher-level system, such as an event bus or a content registry. It is not intended for direct instantiation by clients but is instead returned by a factory or manager class as a "receipt" for a successful registration action.

Its primary architectural purpose is to decouple the registration logic from the unregistration logic. It encapsulates three key concepts:
1.  **Registration State:** An internal boolean flag, `registered`, tracks whether the registration is considered active.
2.  **Conditional Enablement:** A `BooleanSupplier`, `isEnabled`, allows the registration's validity to be dynamically controlled by an external system. A registration is only considered truly active if its internal flag is set *and* the supplier returns true. This is critical for systems like feature flags or mod loading/unloading, where entire sets of registrations can be toggled without being destroyed.
3.  **Encapsulated Cleanup:** A `Runnable`, `unregister`, contains the specific logic required to tear down the registration. This logic is provided by the creating system, ensuring that cleanup is performed in the correct context.

This object acts as a contract between the registering client and the registry system, providing a safe and explicit mechanism for managing the lifecycle of a registered resource.

### Lifecycle & Ownership
-   **Creation:** A Registration object is instantiated and returned by a manager class (e.g., EventManager, BlockRegistry) when a client calls a method like `registerListener` or `registerBlock`. The client code that initiated the registration becomes the owner of the handle.
-   **Scope:** The object's lifetime is controlled by its owner. It should be stored by the owner for as long as the registration needs to be active. If the owner loses the reference to the Registration object, the underlying registration becomes effectively permanent and cannot be explicitly cleaned up, which can lead to memory leaks.
-   **Destruction:** The `unregister` method serves as the explicit trigger for destruction. Calling this method executes the cleanup `Runnable` and internally marks the registration as inactive. The Java object itself is then eligible for garbage collection once all references to it are dropped.

## Internal State & Concurrency
-   **State:** This class is mutable. Its internal `registered` field transitions from `true` to `false` upon the first call to `unregister`. This is a one-way state change.
-   **Thread Safety:** **This class is not thread-safe.** The `unregister` method contains a check-then-act sequence (`if (this.registered ...)` followed by `this.registered = false`). If multiple threads call `unregister` concurrently on the same instance, the encapsulated `unregister.run()` `Runnable` may be executed multiple times, leading to undefined behavior, resource corruption, or crashes.

    **WARNING:** All interactions with a Registration object, particularly calls to `unregister`, must be externally synchronized if it can be accessed from multiple threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| unregister() | void | O(1) | Executes the unregistration logic if the registration is currently active and enabled. This action is idempotent regarding its internal state but not for the encapsulated `Runnable`. |
| isRegistered() | boolean | O(1) | Returns true only if `unregister` has not been called AND the `isEnabled` supplier returns true. This provides the real-time status of the registration. |

## Integration Patterns

### Standard Usage
The owner of a resource or listener stores the returned Registration handle and calls `unregister` during its own shutdown or cleanup phase. This ensures a clean and predictable lifecycle.

```java
// A class that manages a feature that needs to listen to game events
class FeatureManager {
    private Registration gameTickRegistration;

    public void enable(EventManager eventManager) {
        // Register a listener and store the handle
        this.gameTickRegistration = eventManager.register(GameTickEvent.class, this::onGameTick);
    }

    public void disable() {
        // Use the handle to guarantee cleanup of the specific listener
        if (this.gameTickRegistration != null) {
            this.gameTickRegistration.unregister();
            this.gameTickRegistration = null;
        }
    }

    private void onGameTick(GameTickEvent event) {
        // ... feature logic
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Fire-and-Forget:** Calling a registration method and immediately discarding the returned handle. This creates an anonymous, permanent registration that cannot be cleaned up, resulting in memory and logic leaks.
-   **Direct Instantiation:** Do not use `new Registration()`. The `Runnable` and `BooleanSupplier` are deeply tied to the state of the creating registry. Bypassing the registry's factory method will result in a non-functional handle that does not control any real resource.
-   **Concurrent Unregistration:** Calling `unregister` from multiple threads without external locking. This can cause the cleanup logic to run multiple times, which is a severe race condition.

## Data Pipeline
The Registration object is not part of a data processing pipeline. Instead, it functions as a **Control Handle** for managing the lifecycle of other components that may participate in a pipeline.

> Flow:
> Client Code -> Registry.register(callback) -> **Registration Handle** (returned to Client) -> Client holds handle -> Client calls unregister() -> **Registration Handle** executes callback -> Registry is cleaned up

