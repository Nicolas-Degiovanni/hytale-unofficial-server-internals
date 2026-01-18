---
description: Architectural reference for ActionFlockJoin
---

# ActionFlockJoin

**Package:** com.hypixel.hytale.server.flock.corecomponents
**Type:** Transient Command

## Definition
```java
// Signature
public class ActionFlockJoin extends ActionBase {
```

## Architecture & Concepts
ActionFlockJoin is a concrete behavioral primitive within the server-side NPC Action System. It encapsulates the logic for an entity to form or join a social group, known as a flock. This class does not manage flocking behavior itself (such as movement), but rather orchestrates the critical state change of associating one entity with another under a shared flock entity.

As a subclass of ActionBase, it is designed to be a modular, single-purpose command used within a larger AI framework, such as a Behavior Tree or a Finite State Machine. It acts as a high-level controller, delegating the low-level entity-component manipulations to dedicated subsystems like FlockMembershipSystems and FlockPlugin.

The core logic determines the flocking status of both the actor and its target and then executes one of three strategies:
1.  Join an existing flock (either the actor's or the target's).
2.  Create a new flock and have both entities join it.
3.  Do nothing if a compatible flock relationship already exists.

The decision of who becomes the leader of a new flock is determined by the actor's Role configuration, specifically the isCanLeadFlock property.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the NPC behavior system via its corresponding builder, BuilderActionFlockJoin. This typically occurs during the initialization of an NPC's AI definition, where a sequence of potential actions is defined.
-   **Scope:** The object's lifetime is tied to its context within an NPC's active behavior. It is a short-lived, transient object that exists only to be evaluated and potentially executed during a server tick.
-   **Destruction:** The object is eligible for garbage collection as soon as the NPC's AI transitions to a different action or state. It holds no persistent resources and requires no manual cleanup.

## Internal State & Concurrency
-   **State:** The ActionFlockJoin object is stateful, but its configuration is immutable after construction. The primary internal state is the final boolean field forceJoin, set by the builder. The class does not cache any world state between method invocations; all required context is passed in as arguments to the canExecute and execute methods.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to operate exclusively on the main server thread as part of the synchronous game loop tick. Its methods perform direct reads and writes on the EntityStore. Any attempt to invoke its methods from a concurrent thread will result in race conditions, data corruption, and severe world state instability. Synchronization must be handled by the calling AI and entity update systems.

## API Surface
The public contract is inherited from ActionBase and focuses on evaluation and execution within a single game tick.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Precondition check. Returns true if the action can be attempted. Verifies that the actor has a valid target with a known position. |
| execute(...) | boolean | O(1) | Executes the flock joining logic. Modifies the FlockMembership components of the actor and its target. Returns true on successful execution. |

## Integration Patterns

### Standard Usage
This action is intended to be constructed by a builder and executed by the server's NPC processing loop. A developer would typically define this action as part of an NPC's behavioral configuration data.

```java
// In an AI behavior definition (e.g., loaded from data)
BuilderActionFlockJoin builder = new BuilderActionFlockJoin();
ActionFlockJoin joinAction = builder.build();

// During a server tick, the AI system evaluates and runs the action
// The 'ref', 'role', 'sensorInfo', and 'store' are provided by the engine
if (joinAction.canExecute(ref, role, sensorInfo, dt, store)) {
   boolean success = joinAction.execute(ref, role, sensorInfo, dt, store);
   // The NPC's behavior tree would transition based on success
}
```

### Anti-Patterns (Do NOT do this)
-   **Execution Outside Game Loop:** Never call execute or canExecute from an asynchronous task or a different thread. All interactions must be serialized through the main server tick to guarantee safe access to the EntityStore.
-   **Ignoring Preconditions:** Calling execute without a preceding successful call to canExecute in the same tick is unsafe. The execute method relies on the state validations performed by canExecute, such as the existence of a valid target.
-   **State Reuse:** Do not hold a reference to an ActionFlockJoin instance across multiple ticks. It is a stateless command, not a stateful manager. A new evaluation should be performed on each tick.

## Data Pipeline
The flow of data and control for this action involves the sensor, AI, and entity-component systems.

> Flow:
> NPC Sensor System (detects potential flockmate) -> AI Behavior Tree (selects ActionFlockJoin) -> **ActionFlockJoin.execute()** -> FlockMembershipSystems.join() -> EntityStore (writes updated FlockMembership component) -> Other Game Systems (e.g., Movement AI observes new flock state)

