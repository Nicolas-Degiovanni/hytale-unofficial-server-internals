---
description: Architectural reference for AssetPackUnregisterEvent
---

# AssetPackUnregisterEvent

**Package:** com.hypixel.hytale.server.core.asset
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class AssetPackUnregisterEvent implements IEvent<Void> {
```

## Architecture & Concepts
The AssetPackUnregisterEvent is a message-based component within the server's event-driven architecture. It serves as a formal, decoupled signal to notify the system that a specific AssetPack is no longer needed and should be removed from active use.

This class embodies the Command pattern, where an action and its context (the AssetPack to unregister) are encapsulated into a standalone object. By implementing the IEvent interface, it integrates seamlessly with the central EventBus, allowing any part of the server to request an asset pack unregistration without being directly coupled to the AssetManager or other asset-handling subsystems.

The generic type parameter of Void in IEvent<Void> is a critical design choice. It signifies that this is a "fire-and-forget" notification. The publisher of this event does not expect a return value and should not block waiting for a response.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a system component that manages the lifecycle of game content. Common triggers include a world or zone being unloaded, a game mode ending, or a server shutdown sequence initiating.
- **Scope:** Extremely short-lived and transient. The event object exists only for the duration of its propagation through the EventBus.
- **Destruction:** Once all registered listeners have processed the event, it becomes unreachable and is subsequently reclaimed by the Java Garbage Collector. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Immutable. The internal AssetPack reference is declared as final and set only once during construction. This guarantees that the event's payload cannot be mutated while it is being processed by multiple listeners, preventing race conditions and ensuring predictable behavior.
- **Thread Safety:** Inherently thread-safe due to its immutability. An instance of AssetPackUnregisterEvent can be safely published and consumed across different threads without external synchronization.

**Warning:** While the event object itself is thread-safe, the listeners that consume it are responsible for their own thread safety. The AssetManager, for example, must ensure that the unregistration process is properly synchronized to avoid conflicts with other asset operations.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AssetPackUnregisterEvent(AssetPack) | constructor | O(1) | Creates a new event to signal the unregistration of the given AssetPack. |
| getAssetPack() | AssetPack | O(1) | Returns the AssetPack payload associated with this event. |

## Integration Patterns

### Standard Usage
The event should be created and immediately dispatched to the system-wide EventBus. The publisher does not need a reference to the AssetManager or any other listener.

```java
// A world manager determines an asset pack is no longer needed
AssetPack packToUnload = world.getAssociatedAssetPack();

// Fire the event to decouple the world manager from the asset system
// The EventBus will route this to the appropriate listeners.
ServerContext.getEventBus().fire(new AssetPackUnregisterEvent(packToUnload));
```

### Anti-Patterns (Do NOT do this)
- **Direct Handling:** Do not instantiate this event and pass it directly to a manager. This completely bypasses the event bus, breaking the decoupled architecture and preventing other systems (like logging, metrics, or other plugins) from reacting to the event.

  ```java
  // INCORRECT: Bypasses the event bus
  AssetManager am = ServerContext.getAssetManager();
  AssetPackUnregisterEvent event = new AssetPackUnregisterEvent(pack);
  am.handleUnregister(event); // This violates the entire pattern
  ```

- **Payload Modification:** Do not modify the state of the AssetPack object returned by getAssetPack from within a listener. While the event is immutable, the payload object may not be. Listeners should treat the payload as a read-only data structure to avoid unpredictable side effects in other listeners.

## Data Pipeline
The flow of this event through the server is linear and unidirectional, serving as a command from a publisher to one or more subscribers.

> Flow:
> Trigger (e.g., World Unload) -> **new AssetPackUnregisterEvent(pack)** -> EventBus Dispatch -> Listener (AssetManager) -> AssetPack removed from active registry

