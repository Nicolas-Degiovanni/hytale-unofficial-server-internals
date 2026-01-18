---
description: Architectural reference for SingleplayerRequestAccessEvent
---

# SingleplayerRequestAccessEvent

**Package:** com.hypixel.hytale.server.core.modules.singleplayer
**Type:** Transient

## Definition
```java
// Signature
public class SingleplayerRequestAccessEvent implements IEvent<Void> {
```

## Architecture & Concepts
The SingleplayerRequestAccessEvent is not a service or manager, but a message object. It functions as an immutable Data Transfer Object (DTO) specifically designed for the engine's event bus system.

Its sole purpose is to represent a discrete, fire-and-forget request to initiate or join a single-player game session. By implementing the IEvent interface, it signals to the engine that it is a dispatchable message, intended to decouple the request's originator (e.g., a UI component) from its processor (e.g., the SingleplayerModule).

The generic type parameter, Void, is significant. It indicates that handlers subscribing to this event are not expected to return a value. The event is a command, not a query.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by any system component that needs to trigger the single-player access flow. This is typically a UI handler responding to a user clicking "Play" or a console command processor. The creator is responsible for supplying the necessary Access data payload.
- **Scope:** Extremely short-lived. An instance of this event exists only for the duration of its dispatch through the event bus. It is a transient data carrier.
- **Destruction:** Once the event bus has notified all registered listeners, no strong references to the event object remain. It becomes immediately eligible for garbage collection. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Immutable. The internal Access object is stored in a final field, set exclusively at construction time. This guarantees that the event's payload cannot be mutated while it traverses the event bus, preventing unpredictable side effects between different event listeners.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. An instance can be safely created on one thread (e.g., the UI thread) and consumed on another (e.g., a game logic thread) without any need for external synchronization or locks.

## API Surface
The public contract is minimal, exposing only the data payload.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAccess() | Access | O(1) | Retrieves the immutable access details for the single-player session request. |

## Integration Patterns

### Standard Usage
The correct pattern is to create a new instance of the event and post it to the global or context-specific event bus. A dedicated module will listen for this event type and handle the subsequent logic.

```java
// Example: A UI button handler initiating a single-player game
Access playerAccess = Access.newBuilder().build(); // Populate with player data
IEventBus eventBus = context.getService(IEventBus.class);

// Post the event to the bus. Do not hold a reference to it.
eventBus.post(new SingleplayerRequestAccessEvent(playerAccess));
```

### Anti-Patterns (Do NOT do this)
- **Direct Handler Invocation:** Do not instantiate the event and pass it directly to a known handler. This creates tight coupling and bypasses the entire event-driven architecture, defeating the purpose of the class.
- **Event Re-use:** Do not cache and re-post an instance of this event. Each request to access a single-player world must be a new, distinct event object to ensure state integrity.
- **State Inspection:** Do not use this event to query the state of the single-player module. It is a one-way command, not a request for information.

## Data Pipeline
This event acts as a data carrier in a decoupled system. Its flow is linear and unidirectional.

> Flow:
> User Input (UI) -> Handler Logic -> **SingleplayerRequestAccessEvent** (Instantiation) -> Event Bus Dispatch -> SingleplayerModule (Listener) -> World Initialization Sequence

