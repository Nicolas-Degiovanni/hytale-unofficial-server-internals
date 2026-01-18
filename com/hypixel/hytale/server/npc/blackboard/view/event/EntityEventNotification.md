---
description: Architectural reference for EntityEventNotification
---

# EntityEventNotification

**Package:** com.hypixel.hytale.server.npc.blackboard.view.event
**Type:** Transient

## Definition
```java
// Signature
public class EntityEventNotification extends EventNotification {
```

## Architecture & Concepts
The EntityEventNotification is a specialized, transient data-transfer object used within the server-side NPC Artificial Intelligence framework. It functions as a message payload, designed to convey information about a group of entities to an AI's blackboard system.

As a subclass of EventNotification, it integrates into a broader, generic event propagation system. Its specific purpose is to signal changes related to an EntityStore, which typically represents a dynamic collection of entities such as a flock of birds, a pack of wolves, or a squad of soldiers. When an AI agent needs to be made aware of or react to the state of an entire group, this notification is the primary mechanism.

Its placement in the `blackboard.view.event` package indicates its role in updating an AI's "view" of the world—the data structures on its blackboard that model the current game state and drive decision-making processes.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by high-level AI sensory systems or world-state monitors when a significant event concerning a group of entities occurs. For example, a system that manages NPC grouping might create this notification when a new flock is formed.
- **Scope:** Extremely short-lived. An instance of EntityEventNotification exists only for the duration of its dispatch and processing by a consumer, such as an AI blackboard. It is a fire-and-forget message.
- **Destruction:** The object becomes eligible for garbage collection immediately after it has been consumed by the target system. No long-term references should be held by either the producer or the consumer.

## Internal State & Concurrency
- **State:** The class is **Mutable**. Its primary state, the flockReference, is set via a public mutator method after instantiation. It is designed to be configured by its creator before being dispatched, at which point it should be treated as immutable.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. All operations—creation, mutation, and consumption—are expected to occur on the main server thread to prevent data races. Passing this object between threads requires external synchronization, which is a deviation from its intended use.

## API Surface
The public contract is minimal, focusing exclusively on data encapsulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFlockReference() | Ref<EntityStore> | O(1) | Retrieves the reference to the associated group of entities. |
| setFlockReference(ref) | void | O(1) | Sets the reference to the entity group. This should only be called once by the creator of the event. |

## Integration Patterns

### Standard Usage
The intended pattern is to create an instance, populate it with the relevant entity group reference, and immediately dispatch it to the appropriate AI blackboard or event bus for processing.

```java
// A world system detects the formation of a new entity group
Ref<EntityStore> newFlockRef = world.getFlockManager().findFlockById(flockId);

// Create and populate the notification payload
EntityEventNotification notification = new EntityEventNotification();
notification.setFlockReference(newFlockRef);

// Dispatch the notification to the relevant AI agent's blackboard
targetAgent.getBlackboard().postNotification(notification);
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not hold onto and modify an EntityEventNotification instance after it has been dispatched. Each distinct event must have a new instance to ensure data integrity and prevent side effects in downstream systems.
- **Delayed Population:** The flockReference should be set immediately after creation. Dispatching a partially-configured notification can lead to NullPointerExceptions or incorrect AI behavior.
- **State Caching:** Consuming systems must not cache the EntityEventNotification object itself. They should extract the data they need (the flockReference) and discard the notification object. The data pointed to by the reference may change, but the event object is ephemeral.

## Data Pipeline
This class acts as a data carrier in a larger AI event pipeline. Its flow is linear and unidirectional, from a world-state observer to an AI decision-making module.

> Flow:
> World State Change (e.g., flock forms) -> AI Sensory System -> **EntityEventNotification (Instance)** -> AI Blackboard -> Behavior Tree / State Machine Logic

