---
description: Architectural reference for ActionFlockLeave
---

# ActionFlockLeave

**Package:** com.hypixel.hytale.server.flock.corecomponents
**Type:** Transient

## Definition
```java
// Signature
public class ActionFlockLeave extends ActionBase {
```

## Architecture & Concepts
ActionFlockLeave is a concrete implementation of the Command Pattern, operating within the server-side NPC artificial intelligence framework. It represents a single, atomic behavior: the removal of an entity from a flocking group.

This class functions as a terminal node in an NPC's behavior tree or state machine. Its primary architectural role is to decouple the high-level AI decision-making (e.g., "flee from danger") from the low-level mechanics of component manipulation within the Entity Component System (ECS). When executed, its sole responsibility is to remove the FlockMembership component from the target entity, effectively severing its connection to the flock's shared logic and signaling to other systems that it is no longer part of that group.

It is a stateless, single-purpose object whose logic is invoked by a parent AI scheduler, such as a Role or a BehaviorTree executor.

### Lifecycle & Ownership
- **Creation:** Instantiated indirectly via its corresponding builder, BuilderActionFlockLeave. This process is typically driven by the server's configuration loader, which parses NPC definition files (e.g., JSON assets) at server startup or during dynamic NPC spawning. Direct instantiation by developers is an anti-pattern.
- **Scope:** The object's lifetime is bound to the NPC's loaded AI configuration. It is a reusable, stateless command object that persists as long as the NPC's behavior definition is in memory.
- **Destruction:** The instance is eligible for garbage collection when the parent NPC's AI definition is unloaded, typically when the NPC is despawned or the zone it inhabits is shut down.

## Internal State & Concurrency
- **State:** ActionFlockLeave is **stateless and immutable** after construction. Its behavior is determined exclusively by the entity context (Ref, Store) passed into its methods during the game tick. It does not cache data or maintain any internal state between executions.
- **Thread Safety:** This class is **not thread-safe** and must only be invoked from the server's main entity update thread. The methods operate on the ECS Store, which is not designed for concurrent access. The engine's AI scheduler guarantees that actions for a given entity are executed serially, preventing race conditions on component data.

## API Surface
The public contract is defined by its parent, ActionBase, and is intended for consumption by the AI scheduling system, not for general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Returns true if the entity is currently a member of a flock. This is a precondition check. |
| execute(...) | boolean | O(1) | Removes the FlockMembership component from the entity. Always returns true upon completion. |

## Integration Patterns

### Standard Usage
This class is not invoked directly. It is configured within an NPC's asset definition and executed by the AI system. A developer's interaction is declarative, not programmatic.

The conceptual flow within the AI engine is as follows:
```java
// Conceptual example of how the AI scheduler uses this action.
// This code does not exist in this form; it is for illustration.

// 1. An AI behavior tree node decides to execute the "leave flock" action.
ActionFlockLeave action = npc.getRole().getAction("LeaveFlockAction");

// 2. The scheduler checks if the action is valid in the current context.
if (action.canExecute(entityRef, role, sensorInfo, dt, worldStore)) {

    // 3. If valid, the scheduler executes the action.
    action.execute(entityRef, role, sensorInfo, dt, worldStore);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ActionFlockLeave()`. The object must be constructed via its builder, which is handled by the asset loading system. Direct creation bypasses necessary configuration.
- **Execution Outside AI Tick:** Calling `execute` from outside the entity's designated update cycle will corrupt ECS state and lead to severe concurrency bugs. All entity component modifications must be sequenced by the main game loop.
- **Stateful Subclassing:** Do not extend this class to add state. Actions in the Hytale AI framework are designed to be stateless command objects. State should be managed in ECS components.

## Data Pipeline
ActionFlockLeave is a terminal point in a data flow that transforms an AI decision into a change in the world state.

> Flow:
> AI Behavior Tree (Decision) -> Role Scheduler (Execution) -> **ActionFlockLeave.execute()** -> Entity Component Store (Mutation) -> FlockMembership Component Removed -> Flocking System (State Update)

