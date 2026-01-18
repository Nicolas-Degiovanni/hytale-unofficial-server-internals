---
description: Architectural reference for ActionAddToTargetMemory
---

# ActionAddToTargetMemory

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.corecomponents
**Type:** Transient

## Definition
```java
// Signature
public class ActionAddToTargetMemory extends ActionBase {
```

## Architecture & Concepts
ActionAddToTargetMemory is a concrete implementation of the ActionBase command pattern, operating within the server-side NPC AI framework. Its sole responsibility is to mutate the state of an NPC's TargetMemory component, effectively making the NPC "remember" a hostile entity it has detected.

This class is a fundamental building block in the NPC Combat Action Evaluator system. It does not make decisions; rather, it is the *result* of a decision made by a higher-level AI construct, such as a Behavior Tree or a Finite State Machine, which is encapsulated within an NPC's Role.

Architecturally, it serves as a bridge between sensory input and NPC memory. An NPC's sensors provide an InfoProvider object containing information about the environment, including potential targets. If the AI logic determines that a target is hostile and should be remembered, it executes this action. The action then uses Hytale's Entity-Component-System (ECS) primitives—specifically Ref and Store—to locate the NPC's own TargetMemory component and write the new target's entity reference into it.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly via their constructor. They are instantiated by a corresponding builder, BuilderActionAddToTargetMemory, which is typically configured via HOCON or JSON asset files that define an NPC's complete behavioral profile. The action is part of a pre-defined set of capabilities loaded with the NPC's Role.
- **Scope:** The object instance persists for the lifetime of the NPC's Role definition. As a command object, it is stateless and can be reused for multiple executions across the NPC's lifespan.
- **Destruction:** The object is subject to standard garbage collection. It is de-referenced and cleaned up when the parent NPC entity is destroyed or its behavioral assets are unloaded by the server. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** This class is stateless. All configuration is injected via the builder during its construction and is immutable thereafter. The execute method is a pure function with respect to the class instance, though it produces side effects by modifying the state of the TargetMemory component.
- **Thread Safety:** This class is **not thread-safe** and must not be treated as such. All NPC AI logic, including the execution of actions, is designed to run exclusively on the main server game-tick thread. Calling execute from any other thread will violate the engine's concurrency model and lead to component data corruption, race conditions, and server instability.

## API Surface
The public contract is defined by its parent, ActionBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Precondition check. Returns true if the sensory input is valid and provides a target. |
| execute(...) | boolean | O(1) | Executes the core logic. Modifies the TargetMemory component of the source entity. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is integrated into the game engine via NPC configuration files. The AI system, governed by the NPC's Role, selects and executes this action based on runtime conditions.

A typical scenario involves an NPC's sensor detecting a player. The AI's decision-making logic would then trigger this action to add the player to its list of known hostiles.

```java
// Conceptual example of how the AI system might use this action.
// This code would exist within the core AI loop, not in typical gameplay code.

// Assume 'selectedAction' was determined to be an ActionAddToTargetMemory instance
if (selectedAction.canExecute(npcRef, role, sensorInfo, dt, store)) {
    selectedAction.execute(npcRef, role, sensorInfo, dt, store);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new ActionAddToTargetMemory(). The object is not valid unless configured and created by its associated builder, BuilderActionAddToTargetMemory.
- **External Execution:** Do not acquire an instance of this action and call it from outside the server's core NPC update loop. Doing so bypasses the AI's decision-making process and breaks the server's threading model.
- **State Storage:** Do not extend this class with the intent of storing temporary data on it. Action instances are designed to be stateless and reusable.

## Data Pipeline
This component acts as a writer in the NPC's cognitive data flow, persisting a transient sensory perception into long-term component memory.

> Flow:
> NPC Sensor Detection -> InfoProvider Creation -> AI Role Evaluation -> **ActionAddToTargetMemory.execute()** -> Write to TargetMemory Component -> Subsequent AI systems (e.g., Pathfinding, Targeting) read from TargetMemory

