---
description: Architectural reference for PrepareUniverseEvent
---

# PrepareUniverseEvent

**Package:** com.hypixel.hytale.server.core.event.events
**Type:** Transient

## Definition
```java
// Signature
@Deprecated
public class PrepareUniverseEvent implements IEvent<Void> {
```

## Architecture & Concepts
The PrepareUniverseEvent is a message-based component used within the server's core startup sequence. It functions as a Data Transfer Object (DTO) on the central event bus, specifically designed to transport the WorldConfigProvider to various subsystems that must perform initialization before the game universe is fully constructed.

This event is a critical part of an event-driven initialization pattern. Instead of a monolithic bootstrap procedure directly invoking subsystem initializers, the core server fires this event. This decouples the startup orchestrator from the specific subsystems (e.g., world generation, entity management) that require world configuration data.

**Warning:** This class is marked as **Deprecated**. Its use indicates a legacy initialization path. Modern systems should rely on dependency injection or newer, more specific event contracts for configuration distribution. Continued reliance on this event may be unsupported in future versions.

### Lifecycle & Ownership
- **Creation:** Instantiated by a high-level server bootstrap component, such as a ServerEntryPoint or UniverseManager, during the initial phases of server launch. It is created immediately after the primary WorldConfigProvider has been loaded or generated.
- **Scope:** Extremely short-lived. The object's lifecycle is confined to the duration of its dispatch on the event bus. It is created, fired, processed by all synchronous listeners, and then immediately becomes eligible for garbage collection.
- **Destruction:** Managed by the Java Garbage Collector. There are no native resources or explicit cleanup methods associated with this object.

## Internal State & Concurrency
- **State:** Mutable. The event's primary payload, the WorldConfigProvider, can be modified post-construction via the setWorldConfigProvider method. This is a significant design flaw for an event object, as it can lead to unpredictable side effects if one listener modifies the state seen by subsequent listeners.
- **Thread Safety:** **Not thread-safe.** The internal state is exposed via a getter and a setter without any synchronization. If this event were ever dispatched on an asynchronous event bus or accessed by listeners operating in different threads, severe data races could occur. All interactions must be confined to a single, synchronous dispatch thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PrepareUniverseEvent(provider) | constructor | O(1) | Constructs the event, encapsulating the world configuration. |
| getWorldConfigProvider() | WorldConfigProvider | O(1) | Retrieves the encapsulated world configuration provider. |
| setWorldConfigProvider(provider) | void | O(1) | **Warning:** Mutates the event's internal state. Use is strongly discouraged. |

## Integration Patterns

### Standard Usage
This pattern is only valid in legacy code that has not been migrated to newer initialization mechanisms. A system service subscribes to the event and uses the provided configuration to initialize itself.

```java
// Within a listener class (e.g., a WorldInitializer service)
@Subscribe
public void onPrepareUniverse(PrepareUniverseEvent event) {
    WorldConfigProvider config = event.getWorldConfigProvider();
    if (config == null) {
        throw new IllegalStateException("Universe preparation cannot proceed without world config.");
    }
    this.applyWorldConfiguration(config);
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation in Listeners:** Modifying the event payload within a listener is a severe anti-pattern. This creates an invisible, order-dependent coupling between listeners and leads to non-deterministic behavior.

  ```java
  // DO NOT DO THIS
  @Subscribe
  public void onPrepareUniverse(PrepareUniverseEvent event) {
      // Modifying the event breaks the contract for other listeners
      event.setWorldConfigProvider(new CustomTestConfigProvider());
  }
  ```
- **Manual Instantiation and Firing:** This event is part of a controlled, internal server startup sequence. Manually creating and firing it can cause re-initialization of critical systems, leading to state corruption or server crashes.
- **Asynchronous Handling:** Do not process this event asynchronously. Its mutable nature makes it inherently unsafe for concurrent access without external locking, which would defeat the purpose of an event-driven design.

## Data Pipeline
The data flow for this event is linear and part of the server's bootstrap process.

> Flow:
> Server Bootstrap -> Load WorldConfigProvider -> **new PrepareUniverseEvent(provider)** -> EventBus.post() -> Subscribed Listeners (e.g., WorldManager) -> Configuration Applied

