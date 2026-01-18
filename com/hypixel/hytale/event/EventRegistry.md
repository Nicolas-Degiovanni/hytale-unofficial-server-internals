---
description: Architectural reference for EventRegistry
---

# EventRegistry

**Package:** com.hypixel.hytale.event
**Type:** Transient

## Definition
```java
// Signature
public class EventRegistry extends Registry<EventRegistration<?, ?>> implements IEventRegistry {
```

## Architecture & Concepts

The EventRegistry is a hierarchical, scoped proxy for registering event listeners. It is not the primary, global event bus. Instead, it acts as a critical control gate and lifecycle manager for a specific context, such as a mod's initialization phase or a game state's lifetime.

Its core design is based on the **Decorator** pattern. Each EventRegistry instance wraps a parent IEventRegistry, adding two key behaviors:
1.  **Precondition Enforcement:** It can be configured with a `precondition` that must be met for any registration to succeed. This is a powerful mechanism to enforce engine constraints, such as only allowing event registration during a specific startup phase. Attempts to register an event when the precondition is false will result in a runtime exception.
2.  **Scoped Registration Tracking:** While it delegates the actual registration logic to its parent, it simultaneously tracks every successful registration within its own internal list (inherited from the base Registry class).

This dual-action design—**Delegate Up, Track Locally**—is fundamental to the engine's modularity. It allows a subsystem to register a batch of listeners and then later tear them all down cleanly by clearing its specific EventRegistry instance, without affecting any other listeners in the parent or global registries.

## Lifecycle & Ownership

-   **Creation:** EventRegistry instances are never created directly by end-users. They are instantiated and provided by a higher-level framework component, such as a `ModLoader` or `GameStateManager`. This parent component is responsible for supplying the correct parent registry and defining the lifecycle precondition.
-   **Scope:** The lifetime of an EventRegistry is strictly tied to its creator. For example, an EventRegistry provided to a mod during its `onInitialize` method is only valid for the duration of that method's execution.
-   **Destruction:** The owning component is responsible for invoking the `clear` method (from the base Registry class) on the EventRegistry instance. This action iterates through all locally tracked `EventRegistration` objects and unregisters them from the parent registry, ensuring zero listener leakage.

## Internal State & Concurrency

-   **State:** The EventRegistry is mutable. Its internal state consists of a collection of `EventRegistration` objects that grows with each call to a `register` method. The reference to the `parent` registry is final and immutable post-construction.
-   **Thread Safety:** This class is **not thread-safe**. The `checkPrecondition` method is designed to gate access to a specific, well-defined lifecycle phase, which is almost always expected to be on the main engine thread. Attempting to register listeners from other threads will likely fail the precondition check or introduce severe race conditions. All registration activity must be synchronized with the engine's lifecycle.

## API Surface

The public API consists entirely of overloaded `register` methods that mirror the IEventRegistry interface. All of these methods follow the same internal logic: check the local precondition, then delegate the registration to the parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(evt) | EventRegistration | O(1) | Internally adds a pre-created registration to the local tracking list. This is the final step after delegation. |
| register(class, consumer) | EventRegistration | O(1) | Registers a standard, synchronous event listener. Delegates to parent and tracks locally. |
| registerAsync(class, function) | EventRegistration | O(1) | Registers an asynchronous event listener. Delegates to parent and tracks locally. |
| registerGlobal(class, consumer) | EventRegistration | O(1) | Registers a listener that bypasses certain event filtering. Delegates to parent and tracks locally. |
| registerUnhandled(class, consumer) | EventRegistration | O(1) | Registers a listener that is only called if an event was not handled by any standard listener. Delegates to parent. |

## Integration Patterns

### Standard Usage

The EventRegistry is provided to a component by the engine framework. The component uses this instance to register all its required listeners within a designated initialization block.

```java
// Inside a method where 'registry' is a provided EventRegistry instance
// Example: ModInitializer.onInitialize(EventRegistry registry)

// This call is gated by the registry's precondition
registry.register(PlayerJoinEvent.class, this::onPlayerJoin);
registry.register(BlockBreakEvent.class, this::onBlockBroken);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new EventRegistry()`. Doing so creates an orphaned registry that is not connected to the main engine event bus. Your listeners will never be called. Always use the instance provided by the framework.
-   **Lifecycle Escape:** Do not store the provided EventRegistry instance in a static field or pass it to another thread to be used later. Its validity is tied to a specific scope and lifecycle phase. Attempting to use it later will throw an exception due to the precondition failing.
-   **Ignoring Return Value:** The `register` methods return an `EventRegistration` object. While not always necessary, this object can be used to manually unregister a specific listener before the entire registry is cleared. Discarding it removes this option.

## Data Pipeline

The EventRegistry is a component of the **listener configuration pipeline**, not the event firing pipeline. It does not handle or route event objects at runtime. Its role is strictly limited to the setup and teardown phases.

> Flow:
> Framework Component -> Creates **EventRegistry** with precondition -> Provides to Subsystem -> Subsystem calls `register` -> **EventRegistry** checks precondition -> **EventRegistry** delegates to Parent Registry -> **EventRegistry** tracks registration locally

