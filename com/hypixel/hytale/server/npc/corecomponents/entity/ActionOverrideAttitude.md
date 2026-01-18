---
description: Architectural reference for ActionOverrideAttitude
---

# ActionOverrideAttitude

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient

## Definition
```java
// Signature
public class ActionOverrideAttitude extends ActionBase {
```

## Architecture & Concepts
The ActionOverrideAttitude class is a concrete implementation of the `ActionBase` contract, representing a single, executable behavior within an NPC's AI system. It functions as a command object whose sole purpose is to temporarily alter an NPC's disposition, or *Attitude*, towards a specific target entity.

This action is a fundamental building block for creating dynamic NPC interactions. For example, it can be used to make a neutral creature temporarily hostile after being attacked, or to make a guard friendly towards a player who has completed a quest.

Architecturally, it sits within the NPC's behavior processing loop, which is likely managed by a Behavior Tree or a Finite State Machine. It is not a standalone service but a piece of configured logic that is instantiated based on an NPC's asset definition. The reliance on a `BuilderActionOverrideAttitude` and `BuilderSupport` for construction is a key pattern, indicating a factory-based approach to assembling complex NPC behaviors from data. The call to `support.requireAttitudeOverrideMemory()` during construction is a form of dependency validation, ensuring the NPC entity is equipped with the necessary components to manage attitude state before this action is even created.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's NPC asset loading pipeline. A corresponding `BuilderActionOverrideAttitude` reads data from an NPC definition file (e.g., a JSON asset) and, with the help of a `BuilderSupport` context, constructs an instance of this class. It is never created directly by game logic.
- **Scope:** The lifetime of its parent NPC's behavior component. It is part of an NPC's static, configured "skill set" and persists as long as the NPC exists in the world.
- **Destruction:** The object is marked for garbage collection when the parent NPC entity is unloaded from the world. There is no manual destruction method.

## Internal State & Concurrency
- **State:** Immutable. The core state, consisting of the target `attitude` and the override `duration`, is set once at construction via the builder and is declared `final`. The action itself does not maintain any mutable state between executions; it is a stateless operator on the world.
- **Thread Safety:** This class is not thread-safe and is not designed to be. All NPC AI logic, including the execution of actions, is expected to occur synchronously on the main server thread for the world in which the NPC resides. Accessing this object from other threads will lead to race conditions and unpredictable world state.

## API Surface
The public contract is defined by its parent, `ActionBase`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Checks if the action can be performed. Returns true only if a valid sensor target with a position is available. |
| execute(...) | boolean | O(1) | Performs the core logic. Retrieves the target from sensor data and instructs the NPC's `WorldSupport` to apply the attitude override for the configured duration. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Instead, they define its behavior within an NPC's asset files. The system then instantiates and executes it as part of the NPC's AI tick. The conceptual flow during an AI update is as follows.

```java
// Conceptual representation of an NPC's AI tick
// This code is not called directly; it is managed by the AI scheduler.

// 1. The AI system determines the current action is ActionOverrideAttitude
ActionOverrideAttitude currentAction = npc.getActiveAction();

// 2. The system checks if it can run
boolean canRun = currentAction.canExecute(npcRef, npcRole, sensorInfo, dt, worldStore);

// 3. If conditions are met, the action is executed
if (canRun) {
   currentAction.execute(npcRef, npcRole, sensorInfo, dt, worldStore);
   // The action signals completion, and the AI moves to the next state.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ActionOverrideAttitude()`. The object is fundamentally dependent on the builder pattern for its configuration and validation. Bypassing the builder will result in an improperly configured object that will fail at runtime.
- **State Caching:** Do not modify this class to cache data between calls to `execute`. Actions are designed to be stateless commands that read from the world, operate, and finish. Storing state internally breaks this paradigm.
- **Asynchronous Execution:** Do not attempt to call `execute` from a separate thread. All modifications to world state must be synchronized with the main server tick.

## Data Pipeline
The data flow for this component involves two distinct phases: configuration and execution.

> **Configuration Flow:**
> NPC Asset File (JSON/XML) -> Asset Loader -> `BuilderActionOverrideAttitude` -> **ActionOverrideAttitude Instance** (held by NPC)

> **Execution Flow:**
> Server Tick -> NPC AI Update -> Sensor System provides `InfoProvider` -> `canExecute` check -> **`execute` method** -> `WorldSupport.overrideAttitude` -> Entity Component System updates world state

