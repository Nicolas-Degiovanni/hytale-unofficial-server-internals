---
description: Architectural reference for IEventCallback
---

# IEventCallback

**Package:** com.hypixel.hytale.server.npc.blackboard.view.event
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface IEventCallback<EventType, NotificationType extends EventNotification> {
   void notify(NPCEntity var1, EventType var2, NotificationType var3);
}
```

## Architecture & Concepts
IEventCallback is a functional interface that defines a generic, stateless contract for handling events originating from the server-side NPC AI system. It serves as a core component of an observer pattern, allowing various game systems to react to changes in an NPC's state without being tightly coupled to the state management logic itself.

This interface is central to the NPC Blackboard system. A Blackboard is a common AI design pattern that acts as a shared memory or knowledge base for an AI agent. The IEventCallback provides the mechanism by which AI behaviors, server logic, or other systems are notified when specific data on an NPC's Blackboard is created, modified, or deleted.

The use of a functional interface promotes a clean, declarative style of event handling, encouraging the use of lambda expressions and method references for concise and readable callback implementations. The generic parameters, EventType and NotificationType, provide strong typing and flexibility, allowing the contract to be reused for a wide array of different events, from simple target acquisition notifications to complex state machine transitions.

## Lifecycle & Ownership
As an interface, IEventCallback itself has no lifecycle. The lifecycle described here pertains to its *implementations*, which are typically lambda expressions or anonymous classes.

- **Creation:** Implementations are created dynamically when a system registers itself as a listener for a specific Blackboard event. This is typically done during the initialization of an AI behavior or an NPC-related controller. The implementation is passed to a registration method, such as `BlackboardView.listen(...)`.

- **Scope:** The lifetime of a callback implementation is tied directly to its subscription. It persists as long as the listener is registered with the event dispatcher. If the owning object (e.g., an AI behavior) is destroyed or deactivated, it is responsible for deregistering its callbacks to prevent memory leaks.

- **Destruction:** An IEventCallback implementation becomes eligible for garbage collection once it is no longer referenced by the event dispatcher (i.e., after being deregistered) and its capturing scope is also eligible for collection.

## Internal State & Concurrency
- **State:** The interface contract is inherently stateless. However, implementations, particularly lambda expressions that capture variables from their enclosing scope, can be stateful.

    **Warning:** Capturing mutable state in a callback can lead to unpredictable behavior and race conditions, especially if the captured variable's lifecycle is not synchronized with the callback's registration. Implementations should be designed to be re-entrant and rely on the state provided via the `notify` method parameters rather than captured external state.

- **Thread Safety:** The interface provides no intrinsic thread safety guarantees. All implementations of IEventCallback should assume they will be executed on the main server tick thread.

    **Warning:** Do not perform blocking operations such as network requests, file I/O, or heavy computation within a callback. Doing so will stall the server's main loop, causing catastrophic performance degradation, commonly known as "server lag". All operations must be non-blocking and complete within a fraction of a single game tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| notify(NPCEntity, EventType, NotificationType) | void | O(N) | The single abstract method defining the callback. Invoked by the event system. Complexity is dependent on the implementation (N). |

## Integration Patterns

### Standard Usage
The primary integration pattern is to provide a lambda expression to an event registration method on an NPC's Blackboard view. This couples the event directly to the logic that needs to execute in response.

```java
// How a developer should normally use this
// Assume 'npc' is a valid NPCEntity instance
BlackboardView blackboard = npc.getBlackboard().getView();

// Register a callback to react when the NPC's target changes
blackboard.listen(TargetChangedEvent.class, (entity, event, notification) -> {
    // 'entity' is the NPC this event pertains to
    // 'event' contains data about the new target
    System.out.println("NPC " + entity.getId() + " changed target to " + event.getNewTarget());
    
    // Trigger a new AI behavior in response
    entity.getAiController().setAggressive(true);
});
```

### Anti-Patterns (Do NOT do this)
- **Blocking Operations:** Never perform long-running or blocking tasks inside the `notify` method. This is the most critical anti-pattern and will halt the server thread.

    ```java
    // DO NOT DO THIS
    blackboard.listen(SomeEvent.class, (entity, event, notification) -> {
        // ANTI-PATTERN: Blocking I/O inside a callback
        File log = new File("npc_events.log");
        Files.write(log.toPath(), "Event occurred".getBytes()); 
    });
    ```

- **Complex Stateful Lambdas:** Avoid implementing complex logic or state management directly within a lambda. This makes the code difficult to test, debug, and reason about. Delegate to a well-defined method on a proper class instead.

    ```java
    // ANTI-PATTERN: Overly complex logic in a lambda
    blackboard.listen(DamagedEvent.class, (entity, event, notification) -> {
        int counter = 0; // This captured state is problematic
        if (event.getDamage() > 10) {
            counter++;
            if (counter > 5 && entity.getHealth() < 20) {
                // ... complex, untestable logic
            }
        }
    });
    ```

## Data Pipeline
The IEventCallback is the terminal stage of the Blackboard event pipeline. It translates a data-centric event into a procedural action within the game world.

> Flow:
> World Interaction -> NPC State is altered -> Blackboard detects change -> Event object is created -> Event Dispatcher invokes registered listeners -> **IEventCallback.notify()** -> AI Behavior is executed or Game State is modified

