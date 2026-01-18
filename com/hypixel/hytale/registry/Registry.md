---
description: Architectural reference for the Registry abstract base class
---

# Registry

**Package:** com.hypixel.hytale.registry
**Type:** Abstract Base Class / State Manager

## Definition
```java
// Signature
public abstract class Registry<T extends Registration> {
```

## Architecture & Concepts
The Registry class is an abstract, foundational component for managing collections of lifecycle-aware objects that conform to the Registration interface. It is not a concrete implementation but rather a blueprint for creating specific registries, such as an EventRegistry or a KeybindRegistry.

Its primary architectural role is to centralize the management of dynamic registrations and provide a consistent mechanism for enabling, disabling, and tracking these registrations. A key feature is the concept of a *precondition*, a state that must be true for the registry to operate, and a *wrapping function*, which allows subclasses to augment the behavior of registered objects, often to inject automatic un-registration logic.

This class decouples the systems that *provide* functionality (e.g., a new UI element registering its event listeners) from the systems that *consume* or manage that functionality.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses of Registry are instantiated by high-level service containers or application entry points. The Registry itself is never instantiated directly due to its abstract nature.
- **Scope:** An instance of a Registry subclass typically persists for a major application lifecycle, such as the duration of a client session or a server instance. Its lifetime is managed by the service container that created it.
- **Destruction:** The `shutdown` method provides a soft-destruction mechanism. It disables the registry, preventing new registrations and causing subsequent calls to `register` to fail. This does not clear existing registrations; it merely gates new ones. Full memory reclamation occurs when the owner of the Registry instance releases its reference.

## Internal State & Concurrency
- **State:** The Registry is highly stateful. Its core state consists of the `registrations` list and the `enabled` boolean flag. This internal collection is mutable, growing and shrinking as objects are registered and unregistered.

- **Thread Safety:** **WARNING:** This class is **not thread-safe**. The internal `registrations` list is not a concurrent collection, and its modification methods (`add`, `remove`) are not synchronized. Concurrent calls to `register` from multiple threads will lead to a `ConcurrentModificationException` or other unpredictable behavior. All interactions with a Registry instance must be externally synchronized or confined to a single thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(T registration) | T | O(1) | Adds a new registration to the internal list. Returns a wrapped version of the registration. Throws `IllegalStateException` if the registry is not enabled. |
| shutdown() | void | O(1) | Disables the registry, preventing any new registrations. |
| enable() | void | O(1) | Enables the registry, allowing new registrations. |
| isEnabled() | boolean | O(1) | Returns the current enabled status of the registry. |
| getRegistrations() | List<BooleanConsumer> | O(1) | Returns an unmodifiable view of the internal registration consumers. |

## Integration Patterns

### Standard Usage
The primary pattern is to obtain a concrete Registry implementation from a service provider and use it to register objects. The returned object from the `register` call should be used for any subsequent interactions, as it may contain injected lifecycle logic.

```java
// Example assumes a concrete EventRegistry subclass exists
EventRegistry eventRegistry = serviceContext.get(EventRegistry.class);

// Create a registration object (e.g., an event listener)
MyEventListener listener = new MyEventListener();

// The returned handle contains the un-registration logic
Registration listenerHandle = eventRegistry.register(listener);

// To clean up, unregister the handle
listenerHandle.unregister();
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Access:** Never call `register` or allow un-registration logic to execute from multiple threads simultaneously without external locking. This will corrupt the internal state of the registry.
- **Post-Shutdown Registration:** Do not attempt to call `register` after `shutdown` has been invoked. The operation is designed to fail with an `IllegalStateException`.
- **Ignoring the Returned Handle:** Do not discard the `Registration` object returned by the `register` method. It is a wrapped handle that is essential for proper un-registration and lifecycle management. Modifying the original object directly may bypass critical logic.

## Data Pipeline
The Registry class functions as a stateful container rather than a data processing pipeline. Its flow is centered on object lifecycle management.

> Flow:
> Service creates a `Registration` object -> Service calls `Registry.register(registration)` -> **Registry** adds an unregister callback to its internal list -> **Registry** uses its `wrappingFunction` to create a handle -> **Registry** returns the handle to the Service -> On `handle.unregister()`, the callback is invoked, removing it from the internal list.

