---
description: Architectural reference for ActionAttack
---

# ActionAttack

**Package:** com.hypixel.hytale.server.npc.corecomponents.combat
**Type:** Transient

## Definition
```java
// Signature
public class ActionAttack extends ActionBase {
```

## Architecture & Concepts
The ActionAttack class is a concrete implementation of the ActionBase interface, representing a single, stateful combat action within the Non-Player Character (NPC) AI framework. It serves as the primary bridge between an NPC's high-level decision-making (e.g., a behavior tree selecting "attack target") and the server's low-level interaction system that handles the mechanics of combat.

This class is not a simple function call; it is a multi-tick state machine that orchestrates the entire lifecycle of an attack sequence:
1.  **Target Acquisition & Aiming:** It consumes targeting information from an AimingData object, which is provided by the NPC's sensory systems.
2.  **Weapon & Interaction Selection:** It dynamically determines the appropriate attack interaction to use based on the NPC's equipped weapon, configuration overrides, or sensory input.
3.  **Pre-Attack Checks:** It performs critical validation, including line-of-sight verification and friendly fire avoidance.
4.  **Execution:** It constructs and queues an InteractionChain with the server's InteractionManager, which ultimately executes the attack's effects (damage, sound, particles).

ActionAttack encapsulates the *logic and timing* of an attack, while the specific *mechanics* (damage values, animations, projectile types) are defined in separate, reusable Interaction assets. This separation of concerns allows designers to create complex combat behaviors by composing different ActionAttack configurations with various Interaction assets.

### Lifecycle & Ownership
-   **Creation:** ActionAttack instances are not created directly. They are instantiated and configured by a BuilderActionAttack during the server's asset loading phase. This process reads NPC definition files (e.g., JSON) and constructs the fully configured action object.
-   **Scope:** An instance of ActionAttack is owned by an NPC's Role. It is created once when the Role is defined and persists for the lifetime of that Role definition. It is effectively a reusable template for a specific type of attack.
-   **Destruction:** The object is eligible for garbage collection when its owning Role is unloaded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** This class is highly mutable and maintains its state across multiple game ticks. Key state variables include `attackReady`, `aimingTimeRemaining`, and `attackInteraction`. This internal state machine is essential for managing timed sequences like aiming delays and attack cooldowns.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be operated exclusively by the main server game thread. All methods, particularly `execute`, assume single-threaded access to the EntityStore and other game components. Any concurrent modification would lead to state corruption and server instability.

    The static `THREAD_LOCAL_COLLECTOR` field is a specific, controlled exception. It uses a ThreadLocal to ensure that the data collector used during interaction chain walking is unique to the calling thread. This is a performance optimization to prevent object reallocation and to safely contain data within a single-threaded operation, not an indication that the class itself can be used concurrently.

## API Surface
The primary contract is defined by its `ActionBase` parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Checks preconditions for starting an attack, such as ensuring no other attack is currently executing. |
| execute(...) | boolean | Stateful | The core state machine. Manages aiming, checks, and interaction queuing over multiple ticks. Returns true when the action is complete. |
| activate(...) | void | O(1) | Called when this action becomes the NPC's active behavior. Claims ownership of the AimingData resource. |
| deactivate(...) | void | O(1) | Called when this action is interrupted or completed. Releases ownership of the AimingData resource. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly in code. Instead, they define its behavior declaratively in NPC asset files. The game engine's AI system is responsible for instantiating and executing the action as part of an NPC's Role.

The conceptual flow within the engine is as follows:
```java
// Engine internal code (conceptual)
// An NPC's behavior tree selects this ActionAttack instance.
ActionAttack selectedAction = npcRole.getActiveAction();

// On the next game tick, the engine executes the action.
// The action may return false for several ticks while it aims.
boolean isComplete = selectedAction.execute(entityRef, npcRole, sensorInfo, deltaTime, store);

if (isComplete) {
    // The behavior tree can now select a new action.
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ActionAttack()`. The object is complex and requires its configuration to be injected by the `BuilderActionAttack` during asset loading. Direct instantiation will result in a non-functional object.
-   **External State Manipulation:** Modifying internal state fields like `attackReady` or `aimingTimeRemaining` from outside the class will break the internal state machine and lead to unpredictable combat behavior.
-   **Concurrent Execution:** Calling any method on an ActionAttack instance from a thread other than the main server thread is strictly forbidden and will cause critical race conditions.

## Data Pipeline
ActionAttack processes sensory data to produce a command for the interaction system. The flow is unidirectional and occurs over several game ticks.

> Flow:
> Sensory System (provides `InfoProvider` with `AimingData`) -> **ActionAttack.execute()** [State Machine: Aiming, LoS Check] -> `InteractionManager.queueExecuteChain()` -> Game World Effect (Damage, etc.)

