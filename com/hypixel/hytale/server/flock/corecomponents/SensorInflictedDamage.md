---
description: Architectural reference for SensorInflictedDamage
---

# SensorInflictedDamage

**Package:** com.hypixel.hytale.server.flock.corecomponents
**Type:** Component

## Definition
```java
// Signature
public class SensorInflictedDamage extends SensorBase {
```

## Architecture & Concepts
SensorInflictedDamage is a specialized component within the server-side NPC Artificial Intelligence framework. It functions as a conditional trigger, designed to detect a specific world state: whether an NPC or its associated flock has recently inflicted damage upon another entity.

Architecturally, this class serves as a bridge between the combat system and the AI decision-making layer (e.g., Behavior Trees or Finite State Machines). It encapsulates the logic for querying damage records, checking team affiliations (via the flock system), and identifying a target. By conforming to the SensorBase contract, it allows AI behaviors to remain decoupled from the specifics of combat mechanics. An AI controller simply queries the sensor via the `matches` method; a positive result can trigger a state transition, such as from *Patrolling* to *Engaging*.

The sensor's output is not the boolean result alone. Its primary deliverable is the populated InfoProvider (specifically, an EntityPositionProvider), which contains the location of the damaged entity. This allows subsequent AI actions, such as a "MoveTo" or "Attack" node, to consume the sensor's findings without direct communication.

## Lifecycle & Ownership
- **Creation:** An instance of SensorInflictedDamage is not created dynamically during gameplay. It is instantiated once during the server's loading phase when an NPC's behavior profile is parsed from configuration. The construction is handled exclusively through its corresponding builder, BuilderSensorInflictedDamage.
- **Scope:** The object's lifetime is bound to the AI controller of the NPC that owns it. It persists as long as the parent NPC entity is active in the world. Each NPC with this sensor in its behavior set will have its own unique instance.
- **Destruction:** The instance is marked for garbage collection when the parent NPC entity is despawned and its AI controller is torn down. No manual destruction or cleanup is required.

## Internal State & Concurrency
- **State:** This component is highly **mutable** and stateful. Its primary internal state is the EntityPositionProvider field, which caches the reference and position of the last valid target found by the `matches` method. This state is transient and is reset or updated on every execution of the AI tick.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and operated by a single NPC's AI controller, which executes on the main server game thread. Any concurrent access from other threads will result in race conditions and undefined behavior, as the internal positionProvider is mutated without any synchronization mechanisms.

    **WARNING:** Do not share instances of this sensor across multiple NPC entities or access them from asynchronous tasks.

## API Surface
The public contract is minimal, adhering to the SensorBase interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates the sensor's condition. Mutates internal state by updating the position provider. Returns true if a valid, non-friendly target that has been damaged is found. |
| getSensorInfo() | InfoProvider | O(1) | Returns the internal position provider, which contains information about the target found during the last successful `matches` call. |

## Integration Patterns

### Standard Usage
This component is not intended to be invoked directly in procedural code. It is configured declaratively as part of an NPC's behavior definition, typically in a JSON or HOCON file, and executed by the AI engine.

A conceptual configuration might look like this:
```yaml
# (Illustrative NPC Behavior Configuration)
behaviorTree:
  root:
    type: Selector
    children:
      - sequence:
          - sensor:
              type: "SensorInflictedDamage"
              target: "Flock"
              friendlyFire: false
          - action:
              type: "ActionPursueTarget" # This action would consume the sensor's InfoProvider
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SensorInflictedDamage()`. The component must be configured and built via its builder, BuilderSensorInflictedDamage, to ensure its parameters (target, friendlyFire) are correctly initialized.
- **Stale State Reading:** Do not call `getSensorInfo()` without first ensuring `matches()` has been called and returned true within the same AI tick. The data in the InfoProvider is only valid for the duration of the tick in which it was populated.
- **Stateful Reuse:** Do not attempt to cache and reuse the boolean result of `matches()` across multiple ticks. The world state can change, and the sensor must be re-evaluated on every tick to provide an accurate assessment.

## Data Pipeline
The flow of information through this component is reactive, triggered by the NPC's AI update loop.

> Flow:
> 1. External combat logic causes damage, updating a **DamageData** component on an NPCEntity or Flock.
> 2. The server's AI engine ticks the NPC's behavior tree.
> 3. The behavior tree executes the **SensorInflictedDamage.matches()** method.
> 4. The sensor reads the **DamageData** component via the entity store.
> 5. If a valid target is found, the sensor populates its internal **EntityPositionProvider**.
> 6. The boolean result informs the behavior tree, potentially activating a new branch.
> 7. A subsequent AI action node calls **getSensorInfo()** to retrieve the target's location from the **EntityPositionProvider**.

