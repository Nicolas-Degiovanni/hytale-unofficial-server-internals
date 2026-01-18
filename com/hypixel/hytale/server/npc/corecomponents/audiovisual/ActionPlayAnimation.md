---
description: Architectural reference for ActionPlayAnimation
---

# ActionPlayAnimation

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual
**Type:** Transient Command Object

## Definition
```java
// Signature
public class ActionPlayAnimation extends ActionBase {
```

## Architecture & Concepts
The ActionPlayAnimation class is a concrete implementation of the **Command Pattern**, representing a single, atomic operation within the server-side NPC AI system. Its sole responsibility is to instruct an NPCEntity component to play a specific animation in a designated animation slot.

This class acts as a bridge between the data-driven NPC behavior definitions and the live entity component system. It is designed to be embedded within higher-level AI constructs, such as a Behavior Tree or a Finite State Machine, which are managed by an NPC's Role. By encapsulating this logic, the system decouples the *intent* to play an animation from the *execution*, allowing AI behaviors to be defined in static data files (e.g., JSON) and instantiated at runtime.

An ActionPlayAnimation object is not a service or a manager; it is a lightweight, stateful command that is executed by the NPC's core update loop.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly. They are instantiated by the NPC asset loading pipeline via a corresponding builder, `BuilderActionPlayAnimation`. This builder reads configuration from an NPC's behavior definition file and constructs the action as part of a larger AI behavior graph.
- **Scope:** The lifetime of an ActionPlayAnimation object is bound to its owning AI component, typically an NPC's `Role` or behavior tree. It is created once when the NPC's definition is loaded and persists as a reusable command object. It is **not** created per-frame or per-execution.
- **Destruction:** The object is marked for garbage collection when the parent NPC's `Role` is unloaded. This typically occurs when an NPC entity is removed from the world or when the server performs a full asset reload.

## Internal State & Concurrency
- **State:** Mutable. The object holds two primary pieces of state:
    1.  `slot`: The target `NPCAnimationSlot`. This is immutable and set at construction time.
    2.  `animationId`: The string identifier for the animation asset. This is mutable via a protected setter, allowing more complex AI behaviors to dynamically change the animation before execution.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and operated exclusively by the server's main game loop thread for a specific NPC. All method calls, including execution and internal state modification, must be synchronized with the primary server tick.

    **Warning:** Accessing or modifying an instance of this class from an asynchronous task or a different thread will lead to severe race conditions, data corruption, and undefined server behavior.

## API Surface
The public contract is minimal, focused entirely on the execution of the command.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, sensorInfo, dt, store) | boolean | O(1) | Dispatches the command to play the configured animation on the target NPCEntity. Always returns true, indicating the action completes instantaneously. |

## Integration Patterns

### Standard Usage
This action is not intended to be invoked directly by developers. It is executed by a parent AI system, such as a behavior tree node or a state within a finite state machine.

```java
// Hypothetical usage within a parent AI behavior node's update method.
// The 'playAnimationAction' field is a pre-configured ActionPlayAnimation instance.

// On entering this AI state, execute the animation command.
boolean actionCompleted = this.playAnimationAction.execute(npcRef, role, sensorInfo, dt, worldStore);

if (actionCompleted) {
    // The command has been sent; the AI can now transition
    // to the next state or wait for the animation to finish.
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Do not attempt to construct this class manually. Its constructor requires a builder that is supplied by the internal asset pipeline. All actions must be defined in data files to ensure system stability.
- **State Mutation Across Ticks:** While `animationId` is mutable, modifying it should only occur as part of a single, synchronous AI state transition. Do not schedule a task to modify the animation ID at a later time, as this can break the determinism of the AI.
- **Execution on Incorrect Entities:** An ActionPlayAnimation instance is configured as part of a specific NPC's behavior graph. Executing it with a `Ref` to a different NPC may produce unexpected visual results or trigger runtime errors.

## Data Pipeline
The flow for this command begins with static data and ends with a network update to clients.

> Flow:
> NPC Behavior Definition (JSON) -> `BuilderActionPlayAnimation` -> **ActionPlayAnimation Instance** -> AI System (e.g., Behavior Tree) -> `NPCEntity.playAnimation()` -> Server-to-Client Network Packet

