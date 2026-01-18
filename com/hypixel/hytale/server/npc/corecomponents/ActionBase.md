---
description: Architectural reference for ActionBase
---

# ActionBase

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Transient

## Definition
```java
// Signature
public abstract class ActionBase extends AnnotatedComponentBase implements Action {
```

## Architecture & Concepts
ActionBase is the abstract foundation for all discrete NPC behaviors within the server-side AI framework. It represents a single, executable task that an NPC can perform, such as moving to a location, attacking a target, or playing an animation. This class is a core primitive of the server's AI instruction set.

It is designed to be extended, with concrete subclasses implementing the specific logic for each unique action. The architectural pattern here separates the *pre-condition check* for an action (canExecute) from its *stateful execution* (execute). This allows the AI's central scheduler, typically a Behavior Tree or State Machine within an NPC's Role, to efficiently poll numerous potential actions each tick without incurring the cost of their side effects.

A key concept introduced by ActionBase is the "once" flag. This provides a built-in mechanism for creating actions that should only run a single time within a given context, which is critical for initialization routines, cinematic triggers, or tutorial steps.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly. They are instantiated via a corresponding BuilderActionBase subclass during the server's boot or world-loading phase. These builders parse static NPC behavior definitions from configuration assets, constructing an immutable graph of potential actions for each NPC type.
- **Scope:** An ActionBase object's lifetime is bound to the NPC entity it belongs to. It is a component of the NPC's "brain" or Role and persists as long as the NPC is active in the world.
- **Destruction:** The object is eligible for garbage collection when its parent NPC entity is unloaded from the world, and all references from the NPC's Role component are cleared.

## Internal State & Concurrency
- **State:** Highly mutable. The internal flags, particularly *triggered* and *active*, are modified by the NPC AI controller during the main game loop. This state represents the action's current status within the AI's decision-making process. The *once* field is effectively immutable after construction.
- **Thread Safety:** **This class is not thread-safe.** All methods must be invoked exclusively from the primary server thread responsible for entity updates. The AI system's integrity relies on the single-threaded, sequential processing of NPC ticks. Unsynchronized access from other threads will result in severe race conditions, corrupted AI state, and unpredictable server behavior.

## API Surface
The public contract is designed for consumption by the internal AI scheduling system, not for general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Determines if the action is eligible to run. Checks the *once* and *triggered* flags. Subclasses add further conditions. |
| execute(...) | boolean | O(1) | Performs the action. The base implementation sets the *triggered* flag. Subclasses must call super.execute() and implement game logic. |
| activate(...) | void | O(1) | A lifecycle callback invoked by the AI scheduler when this action becomes the current focus of the NPC. |
| deactivate(...) | void | O(1) | A lifecycle callback invoked when the AI scheduler switches focus away from this action. |
| clearOnce() | void | O(1) | Resets the *triggered* state, allowing a "once" action to be executed again. Use with extreme caution. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to extend ActionBase to create a new, specific NPC capability. The core AI engine will then manage the lifecycle and execution of this custom action.

```java
// A developer's primary interaction is to create a concrete implementation.
// This action would make an NPC play a greeting animation if it has a target.

public class ActionPlayGreeting extends ActionBase {

    public ActionPlayGreeting(BuilderActionBase builder) {
        super(builder);
    }

    @Override
    public boolean canExecute(@Nonnull Ref<EntityStore> ref, @Nonnull Role role, InfoProvider sensorInfo, double dt, @Nonnull Store<EntityStore> store) {
        // Always check the base condition first.
        if (!super.canExecute(ref, role, sensorInfo, dt, store)) {
            return false;
        }
        // Add custom logic: only execute if the NPC has a sensory target.
        return sensorInfo.hasTarget();
    }

    @Override
    public boolean execute(@Nonnull Ref<EntityStore> ref, @Nonnull Role role, InfoProvider sensorInfo, double dt, @Nonnull Store<EntityStore> store) {
        // CRITICAL: This call manages the 'once' flag state.
        super.execute(ref, role, sensorInfo, dt, store);

        // Implement the action's unique logic.
        role.getAnimationController().play("greet");
        return true; // Return true to indicate successful execution.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Tampering:** Do not manually call methods like `activate`, `deactivate`, or `clearOnce` from external game logic. These are strictly controlled by the NPC's internal AI scheduler. Doing so will desynchronize the action's state from the behavior tree, leading to unpredictable behavior.
- **Omitting Super Calls:** In a subclass, failing to call `super.canExecute()` or `super.execute()` will break fundamental behaviors, most notably the "once" execution guarantee. This can cause actions intended to run once to run every tick.
- **Heavy Computation in canExecute:** The `canExecute` method may be called on many different actions every single tick. It must be lightweight and performant. Expensive checks, like pathfinding or complex world queries, should be deferred to the `execute` method.

## Data Pipeline
ActionBase sits at the end of the AI decision-making pipeline, translating a decision into a world state change.

> Flow:
> Server Game Tick -> Entity Update Loop -> NPC AI Controller -> Behavior Tree Evaluation -> **ActionBase.canExecute()** -> (Returns true) -> **ActionBase.execute()** -> Game World State Mutation (e.g., Position Change, Animation Event)

