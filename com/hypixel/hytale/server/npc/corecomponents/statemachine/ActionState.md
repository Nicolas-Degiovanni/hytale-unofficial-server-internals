---
description: Architectural reference for ActionState
---

# ActionState

**Package:** com.hypixel.hytale.server.npc.corecomponents.statemachine
**Type:** Transient

## Definition
```java
// Signature
public class ActionState extends ActionBase {
```

## Architecture & Concepts
The ActionState class is a fundamental command object within the server-side NPC AI framework. It represents a single, atomic operation: to change the state of an NPC. This class is a concrete implementation of the abstract ActionBase, designed to be executed by the NPC's decision-making engine, such as a Behavior Tree or a Finite State Machine.

Its primary architectural function is to decouple the *decision* to change state from the *mechanism* of changing state. When an NPC's AI determines a state transition is necessary, it executes an instance of ActionState.

A key design feature is the distinction between two state scopes:
1.  **Global State:** A change that affects the NPC's primary state, managed by the top-level StateEvaluator component. This is used for major behavioral shifts like transitioning from *Idle* to *Combat*.
2.  **Component-Local State:** A change that affects the internal state of a specific component attached to the NPC. This allows for more granular, encapsulated behaviors, such as a specific weapon-handling component cycling through its own *Reloading* or *Aiming* states without altering the NPC's main combat state.

ActionState objects are not created dynamically during gameplay. They are instantiated once during the server's asset loading phase from NPC definition files (e.g., JSON), making them part of a pre-compiled, data-driven behavior graph.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the server's asset loading pipeline when parsing an NPC's behavior definition. The constructor is passed a BuilderActionState object, which contains the deserialized data from an asset file, and a BuilderSupport object, which provides contextual information from the asset loader.
-   **Scope:** An ActionState instance is a stateless, reusable command. It persists for the entire lifetime of the NPC's loaded behavior definition. The same instance is executed every time the AI triggers this specific state change for any NPC of that type.
-   **Destruction:** The object is eligible for garbage collection only when its parent NPC behavior definition is unloaded from memory, for instance, when a world shuts down or game assets are reloaded.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields (state, subState, componentLocal, etc.) are final and set only once within the constructor. The object itself carries no mutable state related to a specific NPC instance.
-   **Thread Safety:** The class is inherently thread-safe due to its immutability. However, the execute method operates on mutable, non-thread-safe objects like Role and EntityStore. The server's architecture guarantees that all AI updates for a single NPC entity occur on a single, dedicated thread, preventing race conditions on the state it modifies.

**WARNING:** Calling the execute method from multiple threads for the same NPC entity will lead to state corruption and is not supported.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(ref, role, ...) | boolean | O(1) | Sets the state on the target NPC's Role. Always returns true. The logic branches internally based on the componentLocal flag to target either a specific component's state or the NPC's global state. |
| getInfo(role, holder) | void | O(1) | Populates a ComponentInfo holder with diagnostic information about the configured state change. Used for debugging and admin tools. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by game logic developers. It is exclusively executed by the NPC's core AI processing loop. The system selects an ActionState from a pre-defined behavior graph and executes it as part of an NPC's update tick.

A conceptual example of the *engine's* usage:
```java
// Inside an NPC's AI update loop (conceptual)
ActionBase selectedAction = decisionMaker.chooseNextAction(npc);

if (selectedAction instanceof ActionState) {
    // The engine invokes execute, passing the live context of the NPC
    selectedAction.execute(npcRef, npcRole, sensorInfo, deltaTime, entityStore);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to create an ActionState using `new`. These objects must be defined within the NPC asset files and loaded by the server. Manual creation bypasses the asset pipeline and will result in an object lacking the correct context, such as a valid componentIndex.
-   **State Misconfiguration:** Defining a component-local state change in an asset file for a component that does not exist or does not manage its own state will result in no-op behavior or potential errors if the componentIndex is invalid.

## Data Pipeline
The ActionState serves as the final step in the AI's decision-making process, translating a decision into a concrete state mutation.

> Flow:
> NPC Update Tick -> AI Decision Maker evaluates sensory input -> A behavior containing **ActionState** is selected -> `execute()` is called -> `Role.getStateSupport()` is mutated with new state values -> On subsequent ticks, `StateEvaluator` reads the new state -> NPC behavior changes accordingly.

