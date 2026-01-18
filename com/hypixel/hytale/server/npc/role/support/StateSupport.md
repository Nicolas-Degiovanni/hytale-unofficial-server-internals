---
description: Architectural reference for StateSupport
---

# StateSupport

**Package:** com.hypixel.hytale.server.npc.role.support
**Type:** Transient (State Object)

## Definition
```java
// Signature
public class StateSupport {
```

## Architecture & Concepts

StateSupport is the runtime state machine engine and interaction context for an individual NPC's behavioral Role. It is a foundational component within the server-side NPC framework, responsible for tracking and managing the finite state machine (FSM) that dictates an NPC's actions and behaviors.

This class does not contain the AI logic itself; rather, it provides the mechanism for that logic (typically defined in a Role subclass) to transition the NPC between predefined states. It acts as the single source of truth for an NPC's current behavioral state.

Key architectural responsibilities include:

*   **State Tracking:** Maintains the current integer-based *state* and *subState* for maximum performance during the game tick.
*   **State Translation:** Collaborates with a **StateMappingHelper** to translate between human-readable string identifiers for states (e.g., "patrol.searching") and their optimized integer representations. This decouples the AI logic from the underlying data representation.
*   **Transition Control:** Integrates with an optional **StateTransitionController** to manage the logic *between* state changes. This allows for complex sequences of animations, particle effects, or delays when moving from one state to another, without polluting the core state management.
*   **Interaction Management:** Manages the context of player interactions. It tracks which players are close enough to interact (**interactablePlayers**), which have recently performed an interaction (**interactedPlayers**), and whether the NPC is in a **busyState** that prevents interaction.
*   **Flocking Integration:** Provides methods like flockSetState to broadcast state changes across an entire group of NPCs, enabling coordinated group behaviors.

StateSupport is a granular, per-entity object. Each NPC with a stateful Role will have its own dedicated instance of StateSupport.

### Lifecycle & Ownership

-   **Creation:** Instantiated by a **BuilderRole** during the construction phase of an NPC's Role. It is never created directly. The builder configures it with all necessary state mappings, busy state definitions, and the state transition controller from the NPC's asset files.
-   **Scope:** The lifecycle of a StateSupport instance is tightly coupled to its parent Role object. It persists for as long as the NPC exists and its Role is active.
-   **Destruction:** The object is eligible for garbage collection when its parent Role is destroyed. This typically occurs when an NPCEntity is unloaded from the world or its Role is replaced.

## Internal State & Concurrency

-   **State:** This class is highly mutable by design. Its core purpose is to hold and frequently update the NPC's current state, track nearby players, and manage interaction flags. It caches collections of players and state mappings derived from the initial builder configuration. The `missingStates` set is a mutable cache to prevent spamming logs for invalid state transition attempts.

-   **Thread Safety:** **This class is not thread-safe and must only be accessed from the main server thread.** The Hytale server engine operates on a single-threaded game loop for entity updates. All methods, particularly those accepting a ComponentAccessor, assume they are being called within this synchronous, single-threaded context. Unsynchronized access from other threads will lead to severe race conditions, inconsistent state, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(accessor) | void | O(N) | Main tick method. Cleans up internal player tracking lists. N is the number of interactable players. |
| setState(state, subState, clearOnce, skipTransition) | void | O(1) | The primary low-level method for changing the NPC's state. May trigger the StateTransitionController. |
| setState(ref, state, subState, accessor) | void | O(1) | High-level, name-based state change method. Handles string-to-index lookups. |
| inState(state, subState) | boolean | O(1) | Performs a fast integer comparison to check if the NPC is in the specified state. |
| willInteractWith(playerRef) | boolean | O(1) | Checks if a player can currently interact with the NPC. Involves a HashSet lookup and a state check. |
| setInteractable(entityRef, playerRef, ...) | void | O(1) | Manages a player's ability to interact, potentially adding or removing the Interactable component. |
| flockSetState(ref, state, subState, accessor) | void | O(M) | Propagates a state change to all members of a flock. M is the number of entities in the flock. |
| runTransitionActions(ref, role, dt, store) | boolean | Variable | Delegates to the StateTransitionController to execute actions between states. Complexity depends on the actions. |

## Integration Patterns

### Standard Usage

StateSupport is an internal dependency of a Role. Game logic within a Role subclass will call methods on its StateSupport instance to query or change the NPC's state.

```java
// Inside a custom Role's update method
if (this.stateSupport.inState("patrol", "idle")) {
    // Check for nearby enemies...
    if (enemyIsNearby) {
        // Transition to the "combat" state
        this.stateSupport.setState(getRef(), "combat", "engage", getAccessor());
    }
}

// Run any active state transition logic
this.stateSupport.runTransitionActions(getRef(), this, dt, getStore());
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new StateSupport()`. The object will be unconfigured and will cause NullPointerExceptions. It must be created and configured by a BuilderRole.
-   **Asynchronous State Changes:** Do not modify state from network callbacks or other threads. All state changes must be queued and executed on the main server thread to prevent data corruption.
-   **Ignoring Transition Controller:** Calling `setState` with `skipTransition` set to true can bypass critical animations or logic. This should only be used for internal state corrections or initial setup, such as in the `activate` method.

## Data Pipeline

StateSupport is a central hub in several key data flows for an NPC.

**AI-Driven State Change Pipeline:**
> AI System (e.g., Behavior Tree) -> `Role.updateLogic()` -> **`StateSupport.setState()`** -> `StateTransitionController.initiateStateTransition()` -> Animation & Effect Systems

**Player Interaction Pipeline:**
> Proximity System -> `Role.update()` -> **`StateSupport.setInteractable(true)`** -> `Store.ensureComponent(Interactable)` -> `EntityTrackerSystem` -> Network Packet (ComponentUpdate) -> Client UI Prompt

**Player Input Handling Pipeline:**
> Client Input -> Network Packet -> Server Interaction System -> `Role.onInteract()` -> **`StateSupport.consumeInteraction()`** -> AI System (Reacts to interaction)

