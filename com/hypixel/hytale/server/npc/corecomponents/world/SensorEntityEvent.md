---
description: Architectural reference for SensorEntityEvent
---

# SensorEntityEvent

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorEntityEvent extends SensorEvent {
```

## Architecture & Concepts

The SensorEntityEvent is a specialized sensory component within the Hytale NPC AI framework. Unlike sensors that continuously scan the environment for proximity or line-of-sight, this class operates on a discrete, event-driven model. Its primary function is to detect when specific, pre-defined events have occurred to other entities (Players or NPCs) within a given range.

It acts as a listener or a query mechanism into a world-event messaging system. An NPC equipped with this sensor can react to abstract occurrences such as another entity taking damage, dying, or using a specific item, without needing to directly observe the action.

The core of its design relies on pre-calculated message slots. During the NPC's asset-loading phase, the constructor collaborates with the BuilderSupport service to resolve a specific event type, NPC group, and range into a unique integer identifier for a message channel (playerEventMessageSlot, npcEventMessageSlot). This is a critical optimization that allows for highly efficient event lookups during the game loop, avoiding costly string comparisons or complex queries at runtime.

The sensor's behavior is further refined by its searchType (determining whether to check for events from Players, NPCs, or both) and a flockOnly flag, which constrains detection to events originating only from members of the NPC's own social group or "flock". This enables sophisticated, coordinated group behaviors.

## Lifecycle & Ownership

-   **Creation:** SensorEntityEvent is never instantiated directly. It is constructed exclusively by its corresponding builder, BuilderSensorEntityEvent, during the server's NPC asset loading process. The BuilderSupport context is required to correctly resolve and register the event message slots.
-   **Scope:** An instance of this class is scoped to the specific AI behavior it is a part of. It persists as long as the parent NPC exists and its AI definition remains loaded. It is not a global or session-wide object.
-   **Destruction:** The object is eligible for garbage collection when the NPC is despawned or its AI controller is replaced or destroyed. It does not manage any native resources and has no explicit cleanup method.

## Internal State & Concurrency

-   **State:** The component is stateful but becomes **effectively immutable after construction**. Its configuration, including message slots, range, and the flockOnly flag, is set once by the builder and does not change during its lifetime. The class does not cache query results between ticks.

-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the primary server thread responsible for AI updates. Its methods operate on an Entity Component System (ECS) Store, which is assumed to be managed by a single-threaded game loop. Concurrently calling its methods from multiple threads will lead to race conditions and unpredictable behavior within the underlying ECS data structures. The responsibility for thread-safe event *posting* lies with the PlayerEntityEventSupport component, not this sensor.

## API Surface

The public contract is primarily defined by its superclass, SensorEvent. The key implementations are the target-finding methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPlayerTarget(parent, store) | Ref<EntityStore> | O(1) amortized | Queries the parent entity's event mailbox for a matching event from a player. Returns a reference to the event source or null if none is found. |
| getNpcTarget(parent, store) | Ref<EntityStore> | O(1) amortized | Queries the parent entity's event mailbox for a matching event from another NPC. Returns a reference to the event source or null if none is found. |

**Warning:** The performance complexity assumes the underlying PlayerEntityEventSupport component uses an efficient, queue-based data structure for its message slots.

## Integration Patterns

### Standard Usage

This sensor is configured within an NPC's behavior tree or state machine definition file. At runtime, an AI task or node invokes the sensor to check for relevant world events, using the result to drive decision-making.

```java
// Within an AI Behavior Tree node's update method

// The 'parent' is the Ref to the NPC entity running this AI
// The 'store' is the current world state provided by the game loop
SensorEntityEvent damageSensor = this.getDamageSensor(); // Assume sensor is pre-loaded

// Check for a player who recently took damage nearby
Ref<EntityStore> eventSource = damageSensor.getPlayerTarget(parent, store);

if (eventSource != null) {
    // A player has triggered the event, transition to an "investigate" or "attack" state
    blackboard.put("targetEntity", eventSource);
    return BehaviorStatus.SUCCESS;
}

return BehaviorStatus.FAILURE;
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new SensorEntityEvent()`. The object will be malformed without valid message slots, which can only be provided by the BuilderSupport service during asset loading. This will result in assertion errors or silent failures.
-   **State Caching:** Do not cache the result of getPlayerTarget or getNpcTarget across multiple game ticks. The underlying event system polls and consumes messages, meaning a result is only valid for the current tick. You must call the method on every tick you wish to check for the event.
-   **Cross-Entity Usage:** Do not use a sensor instance configured for one NPC to find targets for another. The methods rely on the `parent` entity's components, specifically its PlayerEntityEventSupport mailbox. Using it with the wrong context will produce incorrect results.

## Data Pipeline

The flow of information that makes this sensor work involves several distinct systems, beginning with an action and ending with an NPC behavior change.

> Flow:
> External Action (e.g., Player takes damage) -> Game Logic creates an EntityEvent -> Event is posted to the PlayerEntityEventSupport component (Mailbox) of nearby NPCs -> **SensorEntityEvent** (on its tick) queries the mailbox -> AI Behavior Tree receives the event source entity -> NPC executes a new action (e.g., moves to investigate)

