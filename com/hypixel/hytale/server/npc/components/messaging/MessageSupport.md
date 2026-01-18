---
description: Architectural reference for MessageSupport
---

# MessageSupport

**Package:** com.hypixel.hytale.server.npc.components.messaging
**Type:** Component Base Class

## Definition
```java
// Signature
public abstract class MessageSupport implements Component<EntityStore> {
```

## Architecture & Concepts
MessageSupport is an abstract base class that establishes the fundamental contract for components managing an NPC's communication capabilities on the server. Within Hytale's Entity-Component-System (ECS) architecture, this component acts as the standardized interface for querying the state of an NPC's predefined message slots.

It is not a concrete implementation but rather a blueprint. Any server-side entity that needs to participate in the NPC messaging system must have a component that extends MessageSupport. This design decouples the systems that *trigger* NPC messages (e.g., AI Behavior Trees, Quest Systems) from the systems that *define* the messages for a specific NPC type. The core responsibility of a MessageSupport implementation is to expose an array of NPCMessage objects, each representing a potential message the NPC can deliver.

The generic parameter `Component<EntityStore>` firmly places this component within the server's world simulation, granting it potential access to the broader entity storage system if required by concrete implementations.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses of MessageSupport are instantiated and attached to an NPC entity. This typically occurs when an entity is created from a prefab or template. The abstract `clone()` method is a critical part of this lifecycle, indicating that components are duplicated from a master prototype when a new NPC is spawned into the world.
- **Scope:** The lifecycle of a MessageSupport component is strictly bound to the lifecycle of the parent NPC entity to which it is attached. It persists as long as the NPC exists in the `EntityStore`.
- **Destruction:** The component is marked for garbage collection when its parent NPC entity is removed from the world. There is no manual destruction method exposed; cleanup is managed by the parent entity's lifecycle.

## Internal State & Concurrency
- **State:** This abstract class is stateless. All state is deferred to and managed by concrete subclasses, primarily through the `NPCMessage[]` array returned by `getMessageSlots()`. The state held by subclasses is inherently **mutable**, as message slots can be activated, queued, and disabled during gameplay.
- **Thread Safety:** This component is **not thread-safe**. All interactions with a component instance are expected to occur on the main server game loop thread. Unsynchronized access from other threads will lead to race conditions and unpredictable behavior, such as inconsistent message queue states. The engine's core update loop is responsible for ensuring serialized access to entity components.

## API Surface
The public contract is minimal, focusing exclusively on querying the state of message slots.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMessageSlots() | NPCMessage[] | O(1) | **Abstract.** Returns the array of all message slots for the NPC. |
| isMessageQueued(int) | boolean | O(1) | Checks if a message at a specific index is currently activated or queued. |
| isMessageEnabled(int) | boolean | O(1) | Checks if a message at a specific index is currently enabled for use. |
| clone() | Component | O(N) | **Abstract.** Creates a deep copy of the component and its internal state. |

## Integration Patterns

### Standard Usage
A server-side system, such as an AI behavior controller, retrieves this component from an entity to make logical decisions. The system does not need to know the concrete type, only the MessageSupport contract.

```java
// A server system checking if an NPC can send its first greeting message
Entity npcEntity = world.getEntity(npcId);
MessageSupport messaging = npcEntity.getComponent(MessageSupport.class);

if (messaging != null && messaging.isMessageEnabled(0) && !messaging.isMessageQueued(0)) {
    // Logic to queue the message for delivery
    queueNpcMessage(npcEntity, 0);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never instantiate this class directly using `new` as it is abstract. Concrete subclasses should only be instantiated by the entity creation or component attachment systems.
- **External State Mutation:** Do not directly modify the contents of the array returned by `getMessageSlots()`. This breaks encapsulation and bypasses any logic within the concrete component. Other systems should use dedicated methods or events to request a change in message state.
- **Ignoring Nulls:** The `getMessageSlots()` method can return null. Code that consumes this component must be robust and handle the null case to prevent `NullPointerException`.

## Data Pipeline
MessageSupport is primarily used for state lookups rather than processing a flow of data. It is a data source for decision-making systems.

> Flow:
> AI Behavior Tree -> Entity Component Query -> **MessageSupport.isMessageQueued()** -> Boolean State -> AI Decision Node

