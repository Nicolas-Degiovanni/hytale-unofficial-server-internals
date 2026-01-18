---
description: Architectural reference for ActionDie
---

# ActionDie

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle
**Type:** Transient

## Definition
```java
// Signature
public class ActionDie extends ActionBase {
```

## Architecture & Concepts
The ActionDie class is a specific, terminal command within the server-side NPC AI framework. It represents the final action an NPC can take in a behavior sequence, programmatically triggering its death.

Architecturally, this class implements the Command Pattern. It encapsulates all information needed to perform the "die" operation, which is invoked by a higher-level AI orchestrator, such as a Behavior Tree or a Finite State Machine, via the execute method.

Its primary function is to attach a DeathComponent to the target entity. Unlike a standard death resulting from combat, this action uses a null damage source and zero damage amount. This design indicates its intended use for **scripted events**, such as an NPC completing its quest objective, a cinematic sequence, or a pre-determined despawn, rather than for handling damage from players or other entities.

Crucially, it notifies the NPC's Role that a terminal state has been reached, effectively halting any further AI processing for that behavior branch.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively through its corresponding BuilderActionDie. This builder is typically configured within an NPC's static definition files (e.g., JSON behavior trees) and materialized by the AI system when the action is scheduled for execution.
- **Scope:** Extremely short-lived. An instance of ActionDie exists only for the duration of a single call to its execute method within a single server tick. It is stateless and not designed to persist across ticks.
- **Destruction:** The object is immediately eligible for garbage collection after the execute method completes. There is no pooling or reuse mechanism for action objects.

## Internal State & Concurrency
- **State:** Immutable. The class itself contains no mutable fields. Its behavior is entirely driven by the context provided in the execute method parameters.
- **Thread Safety:** **Not thread-safe.** This class is designed to be executed exclusively on the main server thread that governs the game loop and entity updates. The execute method mutates the state of the entity store and the NPC's Role, which are not thread-safe structures. Invoking this from any other thread will lead to data corruption, race conditions, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Attaches a DeathComponent to the entity and flags the NPC's Role as terminal. Always returns true. |

## Integration Patterns

### Standard Usage
A developer or designer will not typically interact with this class directly in Java code. Instead, it is declared declaratively as part of an NPC's behavior definition. The AI system is responsible for its instantiation and execution.

*Conceptual Example (in a hypothetical behavior tree definition):*
```yaml
# This is not real code, but illustrates the design pattern.
# An NPC is defined with a sequence of actions.
# The AI engine processes this sequence.

behavior:
  type: sequence
  actions:
    - type: "ActionMoveTo"
      target: "objective_location"
    - type: "ActionPlayAnimation"
      animation: "complete_ritual"
    - type: "ActionDie" # The system instantiates and executes ActionDie here.
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Do not manually construct a BuilderActionDie and create an ActionDie instance in game logic. Actions should only be triggered through the NPC's defined AI behaviors.
- **Misuse for Combat Death:** Do not use ActionDie to process death from player or environmental damage. Doing so will bypass critical systems that rely on a valid Damage source, such as kill attribution, loot generation, and event logging. Use the standard damage application pipeline for all combat-related deaths.
- **Conditional Logic:** Do not attempt to wrap the execution of this action in complex conditional logic. As a terminal action, it should represent an unconditional final state in a behavior branch.

## Data Pipeline
The data flow for ActionDie is initiated by the AI system, not by an external event like a network packet.

> Flow:
> NPC AI Orchestrator (e.g., Behavior Tree) -> **ActionDie.execute()** -> Entity Component Store Mutation (adds DeathComponent) -> Server-Side DeathSystem (processes the component on a subsequent tick)

