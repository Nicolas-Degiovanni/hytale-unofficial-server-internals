---
description: Architectural reference for ActionLockOnInteractionTarget
---

# ActionLockOnInteractionTarget

**Package:** com.hypixel.hytale.server.npc.corecomponents.interaction
**Type:** Transient

## Definition
```java
// Signature
public class ActionLockOnInteractionTarget extends ActionBase {
```

## Architecture & Concepts
The ActionLockOnInteractionTarget is a fundamental component within the server-side NPC AI system. It serves as a state-promotion mechanism, bridging a transient interaction context with the NPC's persistent memory.

In the Hytale AI framework, an NPC's behavior is composed of discrete *Actions*. This specific action's responsibility is to capture a temporary entity reference, known as the InteractionIterationTarget, and store it in a more permanent, indexed slot within the NPC's memory, managed by the MarkedEntitySupport.

This pattern is critical for creating complex, multi-step behaviors. For example, when a player interacts with an NPC, the player's entity is temporarily marked as the InteractionIterationTarget. A behavior tree might then execute this action to "lock" the player into memory slot 0. Subsequent actions, such as "TurnToFaceMarkedEntity" or "PlayAnimationForMarkedEntity", can then operate on the entity in slot 0 without needing to know about the original interaction event. It effectively converts a fleeting event into durable state for the duration of an AI sequence.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly via code. They are instantiated by the NPC asset loading pipeline, driven by the BuilderActionLockOnInteractionTarget. This process typically occurs once when an NPC's behavior configuration is loaded from server assets.
-   **Scope:** The object's lifetime is bound to the NPC's behavior configuration. It persists as a node within the AI logic (e.g., a Behavior Tree) for as long as that NPC's Role is active.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC entity is unloaded or its AI configuration is replaced. There is no manual destruction method.

## Internal State & Concurrency
-   **State:** The internal state of this class is **Immutable**. The primary field, targetSlot, is final and set only during construction. The action itself does not cache or modify its own state during execution.
-   **Thread Safety:** This class is **Not Thread-Safe** and must not be accessed from multiple threads. It is designed to be executed exclusively by the single-threaded server game loop that ticks the NPC's AI. Its methods operate on mutable, non-thread-safe components like Role and EntityStore, and all such operations are predicated on the assumption of a single writer.

## API Surface
The public contract is inherited from ActionBase and invoked by the NPC's AI scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Returns true if a valid InteractionIterationTarget exists in the NPC's current state. |
| execute(...) | boolean | O(1) | Retrieves the InteractionIterationTarget and writes its reference to the MarkedEntitySupport at the configured targetSlot. |

## Integration Patterns

### Standard Usage
This action is not intended to be invoked directly. It is configured within an NPC's behavior assets and executed automatically by the AI system. The following conceptual example illustrates the *effect* of the action within an AI tick.

```java
// CONCEPTUAL: The AI system invokes this action during an NPC's update.

// Pre-condition: An interaction has occurred, setting a temporary target.
// role.getStateSupport().setInteractionIterationTarget(playerEntityRef);

// The AI scheduler finds and runs the ActionLockOnInteractionTarget instance.
boolean success = currentAction.execute(npcRef, role, sensorInfo, dt, store);

// Post-condition: The player is now stored in the NPC's memory.
// Subsequent actions can now access this reference.
Ref<EntityStore> lockedTarget = role.getMarkedEntitySupport().getMarkedEntity(0);
// assert lockedTarget == playerEntityRef;
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ActionLockOnInteractionTarget()`. The object's configuration is tied to the asset pipeline and its corresponding builder. Direct creation bypasses this and will result in an unconfigured, non-functional component.
-   **External Invocation:** Do not call the execute method from outside the NPC's AI update loop. Doing so will bypass critical state checks and can corrupt the NPC's memory and behavior.
-   **State Assumption:** Do not chain this action without a preceding trigger that reliably sets the InteractionIterationTarget. While the canExecute method provides a safeguard, architecturally this action depends on a prior event or state.

## Data Pipeline
This component acts as a state mutator rather than a data processor. Its flow is one of control and state promotion.

> Flow:
> Player Interaction Event → `Role.StateSupport` (temporary target is set) → AI Scheduler executes **ActionLockOnInteractionTarget** → `Role.MarkedEntitySupport` (target is persisted to a memory slot) → Subsequent AI Actions read from `MarkedEntitySupport`

