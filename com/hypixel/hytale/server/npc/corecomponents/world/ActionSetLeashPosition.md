---
description: Architectural reference for ActionSetLeashPosition
---

# ActionSetLeashPosition

**Package:** com.hypixel.hytale.server.npc.corecomponents.world
**Type:** Transient

## Definition
```java
// Signature
public class ActionSetLeashPosition extends ActionBase {
```

## Architecture & Concepts
ActionSetLeashPosition is a concrete command object within the server-side NPC Behavior System. It functions as a single, atomic operation within an NPC's behavior tree or state machine, such as a Role. Its sole responsibility is to define or update an NPC's "leash point".

A leash point is a spatial anchor (position and orientation) used by higher-level AI systems to constrain an NPC's movement. For example, a guard NPC might have a leash point set to its guard post, preventing it from pursuing targets beyond a certain radius from that point.

This class operates directly on the Entity-Component-System (ECS) by accessing and mutating component data. It can set the leash point to one of two locations, determined at creation time:
1.  The NPC's own current position and orientation.
2.  The position and orientation of a designated target entity, retrieved from the NPC's sensory information.

This action is designed to be stateless and idempotent within a single game tick.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by its corresponding builder, BuilderActionSetLeashPosition. This process is typically driven by the deserialization of an NPC's behavior configuration file (e.g., a JSON definition for a Role). Developers do not instantiate this class directly.
- **Scope:** Extremely short-lived. An instance of this class exists only for the duration of the `execute` method call within a single server tick. It is a fire-and-forget command.
- **Destruction:** The object becomes eligible for garbage collection immediately after its `execute` method completes. It is not retained or referenced by any long-term system.

## Internal State & Concurrency
- **State:** Immutable after construction. The behavior of the action is determined by the final boolean fields `toCurrent` and `toTarget`, which are set in the constructor. The class itself holds no mutable state and performs no caching. Its purpose is to mutate the state of an NPCEntity component.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be invoked exclusively from the main server game loop thread. The underlying ECS operations for getting and setting component data are not atomic and will lead to race conditions, data corruption, or unpredictable behavior if accessed from multiple threads.

## API Surface
The public contract is limited to the `execute` method inherited from ActionBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Modifies the leash data on the NPCEntity component associated with the provided entity reference. Always returns true to signify successful completion of the action. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is configured declaratively within an NPC's behavior definition and executed by the server's AI scheduler. The system internally uses it as follows.

```java
// Conceptual example of how the AI system executes the action.
// This code would exist within an NPC's Role or Behavior Tree processor.

// Action is pre-built from a configuration file
ActionSetLeashPosition action = ...; 

// On a server tick, the system invokes the action for a specific NPC
boolean result = action.execute(npcEntityRef, currentRole, npcSensorInfo, deltaTime, entityStore);

// The NPC's leash point is now updated.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ActionSetLeashPosition()`. The object's configuration is complex and must be handled by its associated builder, BuilderActionSetLeashPosition, as part of the AI definition loading process.
- **Off-Thread Execution:** Calling the `execute` method from any thread other than the primary server tick thread is strictly forbidden. This will bypass all engine-level concurrency controls and lead to critical state corruption.
- **Stateful Reuse:** Do not attempt to hold a reference to an instance of this class and reuse it across multiple ticks or for different NPCs. Each action is configured for a specific context and is considered ephemeral.

## Data Pipeline
The data flow for this action is a direct read-modify-write pattern within the ECS framework.

> Flow:
> NPC Behavior System -> **ActionSetLeashPosition.execute()** -> Read TransformComponent (from Self or Target) -> Write Leash Data to NPCEntity Component -> NPC Movement AI (consumes new leash data on subsequent ticks)

