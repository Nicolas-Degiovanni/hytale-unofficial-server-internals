---
description: Architectural reference for IProcessedEvent
---

# IProcessedEvent

**Package:** com.hypixel.hytale.event
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IProcessedEvent {
```

## Architecture & Concepts
The IProcessedEvent interface defines a fundamental contract within Hytale's event-driven architecture. It establishes a standardized entry point, processEvent, for any object that needs to be processed by a central dispatcher or event bus. This pattern decouples the event producer from the event consumer; the system dispatching the event does not need to know the concrete type of the object, only that it conforms to the IProcessedEvent contract.

This interface is a key enabler for polymorphism in the event system. It allows heterogeneous collections of event handlers or commands to be managed and executed through a single, unified mechanism. The String parameter in the processEvent method suggests a flexible data-driven approach, where a serialized payload or command identifier is passed to the handler for interpretation.

## Lifecycle & Ownership
As an interface, IProcessedEvent itself has no runtime lifecycle. It is a compile-time contract. The lifecycle described below applies to the **concrete classes that implement** this interface.

- **Creation:** Instances of implementing classes are typically created by systems that generate events or commands. For example, a network packet handler might instantiate a specific IProcessedEvent implementation upon receiving a certain packet type.
- **Scope:** The lifetime of an implementing object is determined by its owner. It may be transient, existing only for the duration of a single method call within the event bus, or it could be a long-lived listener registered with a service.
- **Destruction:** The object that creates or holds a reference to an IProcessedEvent implementation is responsible for its garbage collection. In most scenarios, these are short-lived objects that are eligible for garbage collection once the processEvent method completes.

## Internal State & Concurrency
The IProcessedEvent contract itself is stateless. However, the responsibility for managing state and ensuring thread safety lies entirely with the implementing classes.

- **State:** Implementations may be stateful or stateless. A stateful implementation might modify its internal fields based on the payload received in processEvent.
- **Thread Safety:** This contract provides no guarantees of thread safety. Implementations **must** be designed with the assumption that processEvent could be invoked from any thread, including the main game loop, a network thread, or a worker thread. If an implementation modifies shared state, it is responsible for implementing its own synchronization mechanisms (e.g., locks, atomic variables).

**WARNING:** Implementations of this interface that are called from performance-critical threads (e.g., the main render or update loop) should avoid blocking operations or long-running computations to prevent application stutter.

## API Surface
The public contract consists of a single method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| processEvent(@Nonnull String var1) | void | Varies | The primary entry point for event execution. The complexity is entirely dependent on the concrete implementation. Throws NullPointerException if var1 is null. |

## Integration Patterns

### Standard Usage
A class implements the interface to handle a specific type of event. An external system, such as an EventBus or a command queue, then invokes the processEvent method on instances of that class.

```java
// 1. Define a concrete implementation
public class PlayerLoginEvent implements IProcessedEvent {
    private final PlayerData player;

    public PlayerLoginEvent(PlayerData player) {
        this.player = player;
    }

    @Override
    public void processEvent(@Nonnull String eventPayload) {
        // eventPayload might be an empty string or contain additional context
        System.out.println("Processing login for player: " + player.getName());
        // ... logic to handle player login
    }
}

// 2. An event dispatcher processes the event
IProcessedEvent event = new PlayerLoginEvent(somePlayerData);
eventDispatcher.submit(event, "{}"); // The payload is passed here
```

### Anti-Patterns (Do NOT do this)
- **Blocking Implementations:** Do not perform file I/O, blocking network calls, or heavy computation directly within the processEvent method if it can be called on a time-sensitive thread. Offload such work to a separate worker thread.
- **Ignoring the Payload:** While the String payload may sometimes be unused, do not design systems that universally ignore it. It is part of the core contract and may be used by the framework to pass essential context.
- **Assuming Payload Format:** Do not assume the String payload is always in a specific format (e.g., JSON) unless explicitly guaranteed by the specific event source. Implementations should be robust against malformed or unexpected payload formats.

## Data Pipeline
IProcessedEvent serves as a processing node within a larger data or event flow. It is the point where abstract data is converted into concrete game logic execution.

> Flow:
> Event Source (e.g., Network, UI, Game Logic) -> Event Bus / Dispatcher -> **IProcessedEvent.processEvent(payload)** -> Game State Mutation / Further Events

