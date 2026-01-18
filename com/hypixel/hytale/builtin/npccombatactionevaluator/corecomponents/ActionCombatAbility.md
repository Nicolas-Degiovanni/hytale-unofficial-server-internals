---
description: Architectural reference for ActionCombatAbility
---

# ActionCombatAbility

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.corecomponents
**Type:** Transient

## Definition
```java
// Signature
public class ActionCombatAbility extends ActionBase {
```

## Architecture & Concepts
The ActionCombatAbility class is a concrete implementation of the ActionBase, representing a single, executable combat action within the Non-Player Character (NPC) behavior system. It serves as the critical bridge between high-level AI decision-making and the server's low-level interaction mechanics.

Its primary function is to translate a decision made by the CombatActionEvaluator component—such as "perform the 'lunge' attack"—into a verifiable and executable command. This is not a simple fire-and-forget operation; the class orchestrates a complex sequence of pre-condition checks before committing to the action. These checks include aiming validation, line-of-sight verification, relative positioning against a target, and friendly-fire avoidance.

Upon successful validation, ActionCombatAbility does not execute the combat logic directly. Instead, it constructs and enqueues an InteractionChain with the server's InteractionManager. This design decouples the NPC's intent from the mechanics of the interaction itself, allowing the core server systems to handle the animation, physics, and damage effects in a standardized way. The class is fundamentally a stateful validator and command issuer within an NPC's finite-state machine.

## Lifecycle & Ownership
- **Creation:** ActionCombatAbility is not instantiated directly. It is constructed by its corresponding builder, BuilderActionCombatAbility, during the server's asset loading phase. Each instance is configured with unique parameters (e.g., an attack ID) derived from NPC definition files.

- **Scope:** The object's lifetime is tied to the loaded NPC asset definition. It is a reusable, configured component shared by all NPC instances of the same type. However, its *execution context* is transient and specific to a single NPC during a game tick.

- **Destruction:** The object is eligible for garbage collection when its parent NPC asset definition is unloaded, typically during a server shutdown or when game content is reloaded.

- **Activation & Deactivation:** The lifecycle is actively managed by the NPC's Role system.
    - **activate:** Invoked when the behavior tree selects this action as a potential candidate. This method is responsible for "claiming" shared resources, such as the AimingData object, to prevent conflicting actions.
    - **deactivate:** Invoked when the behavior tree deselects this action. It must release any resources claimed during activation. Failure to balance these calls will result in resource contention and erratic NPC behavior.

## Internal State & Concurrency
- **State:** This class is highly stateful and mutable. Its internal state tracks the progress of a single combat execution attempt.
    - **initialised:** A boolean flag for lazy initialization of cached components, such as the DoubleParameterProvider for positioning.
    - **attack:** A String storing the identifier of the current interaction being processed. This is updated on each execution attempt based on input from the CombatActionEvaluator.
    - **cachedPositioningAngleProvider:** A cached reference to a parameter provider to avoid repeated lookups.

- **Thread Safety:** **This class is not thread-safe and must only be accessed from the main server thread.** It is designed to operate within the synchronous, per-entity update tick of the game loop. The heavy use of component assertions (e.g., `assert combatActionEvaluatorComponent != null;`) and reliance on thread-local collectors (`ActionAttack.THREAD_LOCAL_COLLECTOR`) confirms its single-threaded execution model. Any concurrent access would lead to race conditions and severe state corruption.

## API Surface
The public contract is defined by its inheritance from ActionBase and is invoked exclusively by the NPC's internal Role and behavior systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Pre-condition check. Returns true if the NPC's current state and environment permit this action. |
| execute(...) | boolean | O(1) | Attempts to perform the action. Performs all validation and, if successful, queues an InteractionChain. |
| activate(...) | void | O(1) | Lifecycle hook called when the action becomes a candidate for execution. Claims required resources. |
| deactivate(...) | void | O(1) | Lifecycle hook called when the action is no longer a candidate. Releases claimed resources. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers or modders. It is configured via NPC asset files and managed entirely by the NPC's AI framework. The system internally selects an action and drives its lifecycle.

A simplified conceptual flow within the NPC's `Role` update would be:

```java
// Conceptual example of the engine's internal logic
// This code is not meant to be written by a developer.

// 1. The AI system (e.g., a behavior tree) activates this action.
currentAction.activate(npcRole, sensorInfo);

// 2. On a subsequent tick, the system checks if it can run.
if (currentAction.canExecute(npcRef, npcRole, sensorInfo, dt, store)) {
    // 3. If possible, it executes the action.
    boolean success = currentAction.execute(npcRef, npcRole, sensorInfo, dt, store);
    if (success) {
        // Action completed, the AI can now select a new one.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ActionCombatAbility()`. The object will lack a valid ID and other critical context provided by its builder during asset loading, leading to runtime exceptions or undefined behavior.
- **State Tampering:** Do not modify internal fields like `attack` from outside the class. The internal state machine relies on a strict sequence of operations within the `execute` method.
- **Lifecycle Violation:** Calling `execute` without a preceding `activate` call will result in a failure to claim necessary aiming data, causing most combat actions to fail their pre-conditions. Similarly, failing to call `deactivate` will lead to resource leaks.

## Data Pipeline
ActionCombatAbility acts as a processor and validator in the NPC combat data flow. It consumes high-level intent and sensory data, and its primary output is a low-level command for another system.

> Flow:
> 1.  **CombatActionEvaluator** (Component on NPC) decides which attack to use.
> 2.  **InfoProvider** (Service) supplies sensory data like target location, line-of-sight status, and aiming solutions via `AimingData`.
> 3.  **ActionCombatAbility.execute** is called by the NPC's `Role`.
> 4.  The method reads the attack decision and sensory data.
> 5.  It performs validation checks (positioning, aiming, friendly fire).
> 6.  If valid, it constructs an **InteractionChain**.
> 7.  The chain is queued into the **InteractionManager** (Core Server System) for execution.
> 8.  **ActionCombatAbility** calls `completeCurrentAction` on the **CombatActionEvaluator** to signal completion.

