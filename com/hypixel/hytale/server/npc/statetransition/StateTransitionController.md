---
description: Architectural reference for StateTransitionController
---

# StateTransitionController

**Package:** com.hypixel.hytale.server.npc.statetransition
**Type:** Transient

## Definition
```java
// Signature
public class StateTransitionController {
```

## Architecture & Concepts
The StateTransitionController is a critical component within the server-side NPC Artificial Intelligence framework. It acts as a specialized dispatcher that executes a series of defined *actions* when an NPC's state machine transitions from one state to another. Its primary purpose is to decouple the logic of *when* a state change occurs from *what* happens during that change.

This controller is not a generic state machine itself. Instead, it manages the "in-between" logic. For example, when an NPC transitions from an *Idle* state to an *Attacking* state, this controller is responsible for executing actions like playing a battle cry sound, equipping a weapon, or beginning a charge-up animation.

The controller's behavior is defined entirely by configuration data loaded from game assets. During the server's asset loading phase, builder objects like BuilderStateTransitionController are populated from JSON or HOCON files. The StateTransitionController's constructor consumes these builders, constructing a highly optimized, in-memory lookup table. This table maps a pair of states (a *from* state and a *to* state) to a prioritized list of actions to be executed.

The core data structure is a map where the key is a packed integer representing the from-state and to-state, and the value is an IActionListHolder. This design ensures O(1) lookup complexity for initiating a transition sequence at runtime.

### Lifecycle & Ownership
-   **Creation:** An instance of StateTransitionController is created by the NPC asset loading system when an NPC's Role is being constructed. It is instantiated directly via its constructor, which requires a BuilderStateTransitionController and BuilderSupport object. This process is data-driven and occurs before the NPC entity exists in the world.
-   **Scope:** The lifecycle of a StateTransitionController is tightly bound to the Role object that owns it. It persists as long as the parent Role is loaded in memory. Each Role (e.g., "Zombie", "Archer") has its own configured instance of this controller.
-   **Destruction:** The object is eligible for garbage collection when its owning Role is unloaded. It does not manage any native resources and relies on standard Java garbage collection for cleanup. The `unloaded` and `removed` methods are hooks to allow the contained ActionList objects to clean up their own state.

## Internal State & Concurrency
-   **State:** This class is stateful.
    -   **stateTransitionActions:** An Int2ObjectOpenHashMap that stores the mapping from a state transition pair to an executable action list. This map is populated once in the constructor and is treated as immutable thereafter. The call to `trim()` after population optimizes it for read-heavy access.
    -   **runningActions:** A mutable field that holds a reference to the IActionListHolder currently being executed. This field represents the runtime state of the controller, tracking the active transition. It is set by `initiateStateTransition` and cleared when the action sequence completes.

-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread responsible for ticking the associated NPC. The design, particularly the mutable `runningActions` field and the reliance on a `dt` (delta time) parameter in `runTransitionActions`, is fundamentally single-threaded. Unsynchronized access from other threads will lead to race conditions and unpredictable AI behavior.

## API Surface
The public API provides the interface for an external state machine to trigger and manage transition sequences.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initiateStateTransition(int fromState, int toState) | void | O(1) | Prepares the controller to execute actions for the specified transition. Caches the relevant action list for subsequent calls to runTransitionActions. |
| isRunningTransitionActions() | boolean | O(1) | Returns true if a transition action sequence is currently in progress. |
| runTransitionActions(Ref, Role, double, Store) | boolean | O(N) | Executes the current action list. N is the number of actions in the list. Returns true if the sequence is still running, false if it has completed. |
| loaded(Role role) | void | O(N) | Lifecycle callback that propagates the "loaded" event to all configured ActionList objects. N is the number of unique transition definitions. |
| spawned(Role role) | void | O(N) | Lifecycle callback that propagates the "spawned" event to all configured ActionList objects. |
| registerFactories(BuilderManager manager) | static void | O(1) | A static setup method used during server initialization to register the necessary asset builders for this system. |

## Integration Patterns

### Standard Usage
The controller is designed to be driven by a higher-level AI system, such as a Behavior Tree or a Finite State Machine, which determines the NPC's primary state.

```java
// Assumes 'stateMachine' determines a change from OLD_STATE to NEW_STATE
// Assumes 'controller' is the StateTransitionController instance for this NPC's Role

// 1. Initiate the transition
controller.initiateStateTransition(OLD_STATE, NEW_STATE);

// 2. In the main server tick loop, run the transition until it completes
//    The primary state machine should pause while this returns true.
boolean stillTransitioning = true;
while (stillTransitioning) {
    // In a real scenario, this would be called once per game tick
    stillTransitioning = controller.runTransitionActions(entityRef, role, deltaTime, store);
}

// 3. Once complete, the NPC is fully in NEW_STATE and the primary
//    state machine can resume its logic.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new StateTransitionController()`. The controller is useless without being properly constructed from asset data via its specific constructor. It must be built as part of the Role loading process.
-   **Concurrent Modification:** Do not call any methods on this object from any thread other than the NPC's primary update thread.
-   **Ignoring Lifecycle:** Failure to call the `loaded`, `spawned`, `unloaded`, and `removed` methods will prevent the underlying Action objects from initializing or cleaning up correctly, which can lead to memory leaks or null pointer exceptions.
-   **Skipping Initiation:** Calling `runTransitionActions` without a preceding call to `initiateStateTransition` for the current transition will have no effect and indicates a logic error in the calling state machine.

## Data Pipeline
The flow of data from configuration to execution is a key aspect of this system's design.

> Flow:
> 1. Game Asset Files (JSON/HOCON) ->
> 2. BuilderManager & BuilderFactory ->
> 3. BuilderStateTransitionController (In-memory asset representation) ->
> 4. **StateTransitionController** (Constructor consumes builders) ->
> 5. `stateTransitionActions` map (Optimized runtime lookup table) ->
> 6. AI State Machine triggers `initiateStateTransition` ->
> 7. Game Loop calls `runTransitionActions` ->
> 8. ActionList executes, modifying NPCEntity components ->
> 9. Engine renders updated entity state (e.g., new position, animation)

