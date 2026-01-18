---
description: Architectural reference for NPCMessage
---

# NPCMessage

**Package:** com.hypixel.hytale.server.npc.components.messaging
**Type:** Transient

## Definition
```java
// Signature
public class NPCMessage {
```

## Architecture & Concepts
The NPCMessage class is a fundamental data structure, not a service. It functions as a stateful, transient data object within the server-side NPC artificial intelligence framework. Its primary purpose is to encapsulate a directive, a piece of context, or a signal intended for an NPC, coupled with a managed lifespan.

This class acts as a time-sensitive container for AI state. The core architectural feature is the `age` field, which serves as a Time-To-Live (TTL) counter. This TTL is not self-managed; it must be driven externally by a higher-level system—typically the main server tick—that calls the `tickAge` method.

The use of `Ref<EntityStore>` for the `target` is a critical design choice. It provides a layer of indirection, allowing the message to maintain a safe and valid reference to a world entity without preventing that entity from being unloaded or destroyed. This avoids hard references that could lead to memory leaks or dangling pointers to invalid game objects.

## Lifecycle & Ownership
- **Creation:** An NPCMessage is typically instantiated by a parent AI component, such as a behavior tree node or a finite state machine. The presence of a `clone` method strongly suggests a common pattern where pre-configured message templates are defined and then cloned at runtime to create live instances.
- **Scope:** The object's lifetime is explicitly managed and is not tied to the application session. It exists only as long as it is held by an owning AI component and its `age` has not expired. It is a short-lived object by design.
- **Destruction:** The object has no explicit destruction or cleanup method. It becomes eligible for garbage collection as soon as the owning component releases its reference. This typically occurs immediately after `tickAge` returns true or `deactivate` is called, signaling the end of the message's relevance.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its core fields—`activated`, `age`, and `target`—are expected to change during its lifecycle. It effectively represents a micro-state-machine, transitioning from inactive to active and finally to an expired state.
- **Thread Safety:** **This class is not thread-safe.** It is designed with the expectation of being created, modified, and read exclusively by the main server game loop thread. Any concurrent modification from other threads without external locking mechanisms will result in data corruption and unpredictable AI behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tickAge(float dt) | boolean | O(1) | Decrements the message's remaining age. Returns true if the age has expired. This is the primary lifecycle progression method. |
| activate(Ref<EntityStore> target, double age) | void | O(1) | Primes the message for processing. Sets its lifespan and target, and marks it as active. |
| deactivate() | void | O(1) | Immediately marks the message as inactive, effectively ending its processing regardless of remaining age. |
| getTarget() | Ref<EntityStore> | O(1) | Safely retrieves the target entity reference. Returns null if the reference is no longer valid. |
| isInfinite() | boolean | O(1) | Checks if the message is configured to never expire. |
| clone() | NPCMessage | O(1) | Creates a shallow copy of the message. Useful for instantiating from templates. |

## Integration Patterns

### Standard Usage
An NPCMessage is almost always used as a member field within a larger AI component. The owning component is responsible for driving its lifecycle in response to game ticks.

```java
// Within an NPC Behavior component's update method
NPCMessage currentDirective = this.getMessage();

if (currentDirective.isActivated()) {
    // The tickAge method dictates when the message expires
    if (currentDirective.tickAge(deltaTime)) {
        currentDirective.deactivate();
        // Logic to transition to a new AI state follows
    }
}
```

### Anti-Patterns (Do NOT do this)
- **External Modification:** Do not modify the internal state of an NPCMessage from a system that does not own it. State changes should be managed exclusively by the parent AI component.
- **Ignoring `tickAge` Return Value:** The boolean returned by `tickAge` is the primary signal that the message's lifecycle has ended. Failing to check this value will cause the AI to process an expired directive, leading to logic errors.
- **Reusing Without `activate`:** Reusing a message object without calling `activate` to reset its state will lead to unpredictable behavior based on stale data from its previous use.

## Data Pipeline
NPCMessage is not a processing stage in a pipeline; it is the *payload* that is passed between stages of the NPC AI system.

> Flow:
> AI Behavior Tree Node -> **NPCMessage.activate()** -> Stored in NPC Component State -> Server Game Tick -> **NPCMessage.tickAge()** -> AI State Transition Logic

