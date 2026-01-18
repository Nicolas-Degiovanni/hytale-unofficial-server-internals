---
description: Architectural reference for ISensorEntityPrioritiser
---

# ISensorEntityPrioritiser

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface ISensorEntityPrioritiser extends RoleStateChange {
```

## Architecture & Concepts
The ISensorEntityPrioritiser interface defines a contract for the core logic of NPC target selection and threat assessment. It is a critical component within the server-side NPC AI sensory system. This interface embodies a **Strategy Pattern**, allowing different NPC types to implement varied and complex target prioritization logic without altering the core AI sensory loop.

Its primary responsibility is to evaluate a collection of potential target entities and select the single most important one based on a set of rules. These rules are encapsulated within implementations of this interface. For example, a hostile creature might prioritize the closest player, while a passive animal might prioritize a food source or flee from a predator.

By extending RoleStateChange, this interface directly links the sensory input (perceiving entities) to behavioral output (changing the NPC's current Role). A change in the prioritized target, or the lack thereof, can trigger a state transition in the NPC's behavior tree, such as moving from *Idle* to *Attacking* or *Fleeing*.

## Lifecycle & Ownership
- **Creation:** As an interface, ISensorEntityPrioritiser is not instantiated directly. Concrete implementations are instantiated and attached to an NPC entity, typically during the NPC's own creation or through a component-based configuration system.
- **Scope:** The lifecycle of an implementing object is tightly coupled to the NPC entity that owns it. It persists as long as its parent NPC exists in the world.
- **Destruction:** The implementing object is marked for garbage collection when its parent NPC is unloaded or destroyed. No manual cleanup is typically required.

## Internal State & Concurrency
- **State:** This interface defines a stateless contract. Implementations are **strongly encouraged** to be stateless, operating exclusively on the arguments provided to their methods. Caching results within a single `pickTarget` call is acceptable, but maintaining state across multiple game ticks can lead to severe desynchronization and inconsistent behavior.
- **Thread Safety:** Implementations of this interface **must be thread-safe**. The NPC AI system may run on separate threads from the main server tick. The `pickTarget` method is frequently passed data structures like Store that represent a snapshot of the world state to mitigate concurrency issues. Any implementation that modifies shared state must use appropriate synchronization mechanisms.

**WARNING:** Failure to ensure thread safety in an implementation can lead to server instability, deadlocks, and unpredictable NPC behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getNPCPrioritiser() | IEntityByPriorityFilter | O(1) | Returns a filter used to pre-qualify or rank other NPCs as potential targets. |
| getPlayerPrioritiser() | IEntityByPriorityFilter | O(1) | Returns a filter used to pre-qualify or rank players as potential targets. |
| pickTarget(...) | Ref<EntityStore> | O(N) | The core evaluation method. Iterates through potential targets and returns the highest priority entity. |
| providesFilters() | boolean | O(1) | Indicates if this prioritiser provides additional, generic entity filters. |
| buildProvidedFilters(...) | void | O(1) | Populates a given list with the filters exposed by this prioritiser. |

## Integration Patterns

### Standard Usage
This interface is not called directly by general game logic. It is invoked by an NPC's sensory or AI controller component during its update cycle. The controller is responsible for gathering potential targets and passing them to the `pickTarget` method.

```java
// Example from within a hypothetical NPC AI controller
ISensorEntityPrioritiser prioritiser = npc.getComponent(ISensorEntityPrioritiser.class);
Store<EntityStore> nearbyEntities = world.findEntitiesNear(npc.getPosition());

// The prioritiser determines the most important target from the list
Ref<EntityStore> currentTarget = npc.getCurrentTarget();
Ref<EntityStore> newTarget = prioritiser.pickTarget(
    npc.getRef(),
    npc.getCurrentRole(),
    npc.getPosition(),
    currentTarget,
    npc.getLastAggressor(),
    false,
    nearbyEntities
);

// The AI controller then acts on the result
if (!newTarget.equals(currentTarget)) {
    npc.setTarget(newTarget);
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not design an implementation that stores a target or other state across multiple ticks. The AI system expects the prioritiser to be a pure function of its inputs for each tick.
- **Blocking Operations:** The `pickTarget` method must execute quickly. Do not perform file I/O, network requests, or complex, long-running calculations within an implementation. This will block the NPC AI thread and degrade server performance.
- **Ignoring Existing Target:** The `pickTarget` method is passed the NPC's current target. A robust implementation should use this information to provide target "stickiness" and prevent the NPC from rapidly switching between targets of similar priority.

## Data Pipeline
The ISensorEntityPrioritiser sits at the decision-making core of the NPC perception pipeline.

> Flow:
> World Entity Query -> Raw List of Nearby Entities -> **ISensorEntityPrioritiser.pickTarget()** -> A Single, Prioritized Target Entity -> NPC Behavior System (State Machine / Behavior Tree)

