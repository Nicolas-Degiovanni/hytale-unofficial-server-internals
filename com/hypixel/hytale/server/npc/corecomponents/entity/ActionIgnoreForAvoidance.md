---
description: Architectural reference for ActionIgnoreForAvoidance
---

# ActionIgnoreForAvoidance

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient

## Definition
```java
// Signature
public class ActionIgnoreForAvoidance extends ActionBase {
```

## Architecture & Concepts
ActionIgnoreForAvoidance is a concrete implementation of the **Command Pattern** within the server-side NPC Artificial Intelligence framework. It represents a single, atomic operation that an NPC can perform as part of a larger, more complex behavior defined in a Behavior Tree or Finite-State Machine.

Its specific function is to modify an NPC's internal state to temporarily disregard a specific target entity when calculating avoidance and pathfinding vectors. This is a tactical maneuver, allowing an NPC to, for example, move through an ally or ignore a non-threatening entity that is blocking its path to a primary objective.

This action does not perform the avoidance logic itself. Instead, it acts as a state mutator for the NPC's `Role` context. Downstream systems, particularly the movement and pathfinding processors, are responsible for querying this state via the `MarkedEntitySupport` component to determine which entities should be factored into their calculations. The use of a `Builder` for its construction signifies that this action is configured via data assets, not hard-coded, allowing designers to tune NPC behavior without code changes.

## Lifecycle & Ownership
- **Creation:** Instances are created by the server's asset loading pipeline. A corresponding `BuilderActionIgnoreForAvoidance` reads data from an NPC definition file (e.g., a HOCON or JSON asset) and instantiates this action. It is never created directly using the `new` keyword in gameplay logic.
- **Scope:** Extremely short-lived and stateless. An instance of this action typically exists only for the duration of a single server tick in which it is executed. It is a transient command object.
- **Destruction:** The object becomes eligible for garbage collection immediately after its `execute` method returns. It holds no persistent state and is not retained by the AI system.

## Internal State & Concurrency
- **State:** Immutable. The internal state, `targetSlot`, is a final field set only once during construction. The action itself is a pure function definition; all stateful changes are performed on the `Role` object passed into the `execute` method.
- **Thread Safety:** This class is **not thread-safe** in the context of its execution. While the object's internal state is immutable, the `execute` method mutates the state of the passed-in `Role` object. All actions for a given NPC must be executed serially on the world's main server thread to prevent race conditions and AI state corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Mutates the NPC's `MarkedEntitySupport` state to ignore the entity in the configured target slot for avoidance calculations. Always returns true. |

## Integration Patterns

### Standard Usage
A developer or designer does not invoke this action directly. It is defined within an NPC's behavior asset and is executed by the core AI engine when the corresponding node in the Behavior Tree is ticked.

```java
// CONCEPTUAL: Invoked by an NPC's Behavior Tree or FSM processor.
// A developer does not call this method directly.

// 1. An NPC's behavior asset defines a sequence that uses this action.
//    (Defined in a HOCON or JSON file)
//    ...
//    action: "IgnoreForAvoidance",
//    targetSlot: "primary_threat"
//    ...

// 2. During an NPC's update tick, the AI engine executes the action.
Action currentAction = npcBehaviorTree.getCurrentAction();
boolean result = currentAction.execute(entityRef, npcRole, sensorInfo, deltaTime, worldStore);

// 3. The result (always true) causes the Behavior Tree to proceed.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ActionIgnoreForAvoidance()`. This bypasses the data-driven design and will likely fail due to the missing `Builder` and `BuilderSupport` context. All actions must be defined in assets.
- **Stateful Reuse:** Do not attempt to cache and reuse an instance of this action. It is designed to be a transient, fire-and-forget command.
- **Asynchronous Execution:** Calling `execute` from any thread other than the NPC's designated update thread is a critical error. This will lead to severe concurrency issues and unpredictable AI behavior.

## Data Pipeline
This action serves as a control signal that modifies state for consumption by other systems. The data does not flow *through* this class, but rather originates from it.

> Flow:
> NPC Behavior Tree Tick -> **ActionIgnoreForAvoidance.execute()** -> `Role.MarkedEntitySupport` (State Mutation) -> Pathfinding System (Reads State) -> Movement Vector Calculation

