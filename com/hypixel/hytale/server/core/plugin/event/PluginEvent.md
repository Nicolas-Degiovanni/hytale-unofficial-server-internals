---
description: Architectural reference for PluginEvent
---

# PluginEvent

**Package:** com.hypixel.hytale.server.core.plugin.event
**Type:** Base Class

## Definition
```java
// Signature
public abstract class PluginEvent implements IEvent<Class<? extends Plugin-Base>> {
```

## Architecture & Concepts
The PluginEvent class is an abstract base type that serves as the foundation for all events related to the server's plugin lifecycle. It is a critical component of the server's event-driven architecture, establishing a standardized contract for any event that originates from or targets a specific plugin.

Its primary architectural role is to act as a data carrier that immutably binds an event to its source plugin instance (PluginBase). This design ensures that any system listening for plugin-related events has immediate and reliable access to the context of the plugin that triggered the event. This class is not used directly but is extended by concrete event types such as PluginLoadEvent or PluginUnloadEvent, which are then dispatched onto the central server EventBus.

## Lifecycle & Ownership
- **Creation:** Instances of PluginEvent subclasses are created exclusively by the server's internal PluginManager. This occurs in response to significant lifecycle transitions of a plugin, such as its initial loading, enabling, or disabling. Direct instantiation by plugin developers is not supported and is architecturally incorrect.
- **Scope:** An event object is ephemeral and has a very short lifespan. It exists only for the duration of its dispatch cycle within the EventBus. Once all registered listeners have processed the event, it is no longer referenced by the core system and becomes eligible for garbage collection.
- **Destruction:** Managed automatically by the Java Garbage Collector once the event dispatch is complete and no strong references remain. Listeners must not retain references to event objects beyond the scope of their handler method.

## Internal State & Concurrency
- **State:** The state of a PluginEvent is immutable. The core state, a reference to the source PluginBase, is marked as final and is injected via the constructor. This makes the event object a simple and predictable Data Transfer Object (DTO).
- **Thread Safety:** This class is inherently thread-safe due to its immutability. It can be safely read by multiple threads without synchronization.
    - **WARNING:** While the PluginEvent object itself is thread-safe, the contained PluginBase object may not be. Consumers of this event must adhere to the concurrency guarantees of the PluginBase class itself when accessing its state or methods.

## API Surface
The public contract is minimal, focusing entirely on providing access to the source plugin.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PluginEvent(PluginBase) | constructor | O(1) | Internal constructor for subclasses. Requires a non-null plugin instance. |
| getPlugin() | PluginBase | O(1) | Returns the non-null plugin instance associated with this event. |

## Integration Patterns

### Standard Usage
Developers do not interact with PluginEvent directly. Instead, they create listeners for its concrete subclasses. The primary pattern is to receive a specific event, extract the source plugin, and perform context-aware actions.

```java
// A listener for a concrete event that extends PluginEvent
public class MyPluginListener {

    @Subscribe
    public void onPluginLoad(PluginLoadEvent event) {
        // Retrieve the plugin that was just loaded
        PluginBase loadedPlugin = event.getPlugin();

        // Perform actions based on the loaded plugin's context
        System.out.println("Plugin loaded: " + loadedPlugin.getName());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** It is impossible to instantiate `new PluginEvent()` as it is abstract. Furthermore, instantiating its concrete subclasses is an anti-pattern; these events must only be created and dispatched by the server's PluginManager to ensure system integrity.
- **Storing Event References:** Do not cache or store event objects in long-lived collections. They are transient data carriers and should be considered stale immediately after the event handler returns.

## Data Pipeline
The flow of data for any plugin-related event follows a clear, unidirectional path through the server's event system.

> Flow:
> PluginManager State Change -> Instantiates **Concrete PluginEvent** -> Dispatches to EventBus -> EventBus Notifies Listeners -> Listener executes logic using `getPlugin()`

