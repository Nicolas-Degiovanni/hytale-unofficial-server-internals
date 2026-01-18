---
description: Architectural reference for NPCBlockEventSupport
---

# NPCBlockEventSupport

**Package:** com.hypixel.hytale.server.npc.components.messaging
**Type:** Component (ECS)

## Definition
```java
// Signature
public class NPCBlockEventSupport extends EventSupport<BlockEventType, EventNotification> implements Component<EntityStore> {
```

## Architecture & Concepts
The NPCBlockEventSupport is a specialized component within the server's Entity-Component-System (ECS) architecture. Its sole purpose is to enable a non-player character (NPC) entity to perceive and react to block-related events occurring within its vicinity.

Architecturally, it serves as a message-passing bridge. It subscribes the parent NPC entity to the server's global block event stream, filters those events based on relevance (such as proximity), and dispatches them to the NPC's internal logic systems, typically its AI Blackboard or associated behavior trees.

By extending EventSupport, it inherits a robust, generic mechanism for managing event listeners. This class provides the specific implementation for BlockEventType, ensuring that raw world events are translated into meaningful notifications that an NPC's AI can process. As a Component, its lifecycle is intrinsically tied to a parent entity, making it a modular piece of an NPC's overall capabilities.

## Lifecycle & Ownership
- **Creation:** This component is not designed for direct instantiation. It is created by the entity management system, typically by invoking its clone method when an NPC is spawned from a template or archetype. It can also be added programmatically to an existing entity to grant it this capability dynamically.
- **Scope:** The component's lifetime is strictly bound to its parent NPC entity. It exists only as long as the entity is active in the world.
- **Destruction:** It is marked for garbage collection and its internal listeners are cleared when the parent NPC entity is despawned or removed from the world's EntityStore. The ECS framework manages this process automatically.

## Internal State & Concurrency
- **State:** NPCBlockEventSupport is a stateful component. Its primary state is the collection of registered listeners inherited from the EventSupport base class. This collection is mutable, as listeners are added and removed during the NPC's lifetime.
- **Thread Safety:** This component is **not thread-safe**. It is designed to be accessed and modified exclusively by the server's main world-update thread that ticks the parent entity. Unsynchronized access from other threads will lead to collection modification exceptions and severe state corruption.

## API Surface
The primary API is inherited from the EventSupport base class for registering and unregistering listeners.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| clone() | Component | O(N) | Creates a new instance of the component, copying all existing listener registrations. N is the number of listeners. |
| listen(type, callback) | void | O(1) | (Inherited) Registers a callback to be executed when a specific BlockEventType occurs. |
| unlisten(type, callback) | void | O(1) | (Inherited) Deregisters a previously registered callback. |

## Integration Patterns

### Standard Usage
The intended use is for an NPC's AI system (e.g., a behavior tree node or a script) to retrieve this component from its parent entity and subscribe to specific block events to trigger behaviors.

```java
// Within an NPC's AI logic, 'self' is the parent entity
NPCBlockEventSupport eventSupport = self.getComponent(NPCBlockEventSupport.getComponentType());

// Make the NPC react when a player breaks a nearby log
eventSupport.listen(BlockEventType.BLOCK_BREAK, (notification) -> {
    if (notification.getBlock().isType("log")) {
        self.getAI().setTarget(notification.getSourcePlayer());
    }
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new NPCBlockEventSupport()`. The component will not be registered with an entity or the event system, rendering it useless and prone to causing null pointer exceptions.
- **State Caching:** Do not cache a reference to this component across different entities. Each component instance is unique to its parent entity. Attempting to use a cached reference from another NPC will cause events to be routed incorrectly.

## Data Pipeline
This component sits in the middle of the server's event processing pipeline, translating low-level world changes into high-level AI stimuli.

> Flow:
> World Engine (Block is broken) -> Global Block Event Stream -> Spatial Query (Finds nearby entities) -> **NPCBlockEventSupport** (Filters and dispatches) -> NPC Blackboard -> AI Behavior Tree (Triggers reaction)

