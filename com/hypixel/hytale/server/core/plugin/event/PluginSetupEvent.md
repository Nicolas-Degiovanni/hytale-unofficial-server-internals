---
description: Architectural reference for PluginSetupEvent
---

# PluginSetupEvent

**Package:** com.hypixel.hytale.server.core.plugin.event
**Type:** Transient

## Definition
```java
// Signature
public class PluginSetupEvent extends PluginEvent {
```

## Architecture & Concepts
The PluginSetupEvent is a lifecycle signal within the server's event-driven plugin architecture. It does not contain complex logic itself; rather, it serves as a formal notification that a specific plugin has been loaded by the PluginManager and is now entering its initialization phase.

This event is a critical synchronization point. It is broadcast by the server core to allow the plugin itself, as well as other dependent systems, to perform initial setup tasks. These tasks typically include registering command handlers, subscribing to other game events, or establishing connections to other services. It represents the first opportunity for a plugin to execute its own code within the server environment.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the core PluginManager during the bootstrap sequence for each individual plugin. A new event object is created for every plugin that is loaded.
- **Scope:** Extremely short-lived. The event object exists only for the duration of its dispatch through the server's EventManager.
- **Destruction:** Once all registered listeners have processed the event, it falls out of scope and is immediately eligible for garbage collection. No system maintains a long-term reference to event instances.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal state consists of a single reference to the PluginBase instance that is being set up. This reference is assigned during construction and cannot be changed.
- **Thread Safety:** This class is inherently thread-safe due to its immutable nature. It is safe to read the event data from any thread.

    **WARNING:** While the event object is thread-safe, the listeners that consume it are not guaranteed to be. Developers handling this event must ensure their own listener implementations are thread-safe if the EventManager dispatches on multiple threads.

## API Surface
The public contract is inherited from the parent PluginEvent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPlugin() | PluginBase | O(1) | Retrieves the plugin instance that is undergoing setup. |

## Integration Patterns

### Standard Usage
The canonical use case is to create a listener method within a plugin or core service that subscribes to this event. This allows for decoupled initialization logic that executes at a precise moment in the server lifecycle.

```java
// Inside a listener class registered with the EventManager
public void onPluginSetup(PluginSetupEvent event) {
    PluginBase plugin = event.getPlugin();
    Logger.info("Plugin " + plugin.getName() + " is now setting up.");

    // Register commands, other listeners, or perform one-time initialization
    server.getCommandRegistry().register(plugin, new MyPluginCommand());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of PluginSetupEvent manually. This event is part of a managed lifecycle controlled by the server core. Firing a synthetic event will disrupt the expected plugin loading sequence and lead to unpredictable behavior or server instability.
- **Stateful Listeners:** Avoid storing the PluginSetupEvent object itself within a listener. Extract the necessary data (the PluginBase reference) and discard the event object. Holding a reference to the event can prevent garbage collection and serves no practical purpose.

## Data Pipeline
PluginSetupEvent acts as a control signal rather than a data-processing component. Its flow through the system is a broadcast from a central authority to multiple subscribers.

> Flow:
> PluginManager begins loading a plugin -> **PluginSetupEvent is instantiated** -> EventManager dispatches event -> Registered Listeners (including the plugin itself) receive the event -> Listeners execute setup logic

