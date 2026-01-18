---
description: Architectural reference for SensorOr
---

# SensorOr

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Component / Transient

## Definition
```java
// Signature
public class SensorOr extends SensorMany {
```

## Architecture & Concepts
The SensorOr class is a fundamental component within the Hytale NPC Artificial Intelligence framework, specifically serving as a logical compositor for AI perception. It embodies the **Composite Pattern**, acting as a non-terminal node in a behavior tree that can contain multiple child Sensor objects.

Its primary function is to act as a logical **OR** gate. An NPC's AI uses Sensors to perceive the game world—detecting players, checking health thresholds, or identifying environmental conditions. SensorOr allows for the creation of complex, multi-faceted conditions by evaluating a list of child Sensors and succeeding if *any one* of them succeeds.

For example, an NPC might be configured to enter a "flee" state if it sees a player with a weapon *OR* if its health is below 25%. SensorOr is the mechanism that enables this compound logic. It is a core building block for creating nuanced and responsive NPC behaviors, evaluated during the AI's decision-making phase in the main server tick.

## Lifecycle & Ownership
-   **Creation:** SensorOr instances are not created directly via code during gameplay. They are instantiated by the server's asset loading system, specifically through a corresponding builder class, BuilderSensorOr. This occurs when an NPC's behavior asset (e.g., a JSON or HOCON file) is parsed and its behavior tree is constructed in memory.
-   **Scope:** The lifetime of a SensorOr instance is tied to the lifetime of the NPC behavior definition it is part of. It is effectively a static data component of a loaded NPC asset, persisting as long as that asset is held in memory by the server.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC behavior asset is unloaded. This typically happens during a server shutdown, a world unload, or a dynamic asset reload.

## Internal State & Concurrency
-   **State:** SensorOr holds two primary pieces of state:
    1.  An immutable list of child Sensor objects, provided at construction.
    2.  A mutable WrappedInfoProvider instance, inherited from its parent. This provider caches information about which child Sensors matched during an evaluation. Its state is cleared at the beginning of every call to the matches method.

-   **Thread Safety:** **This class is not thread-safe.** The internal WrappedInfoProvider is mutated during the execution of the matches method. Sharing a single SensorOr instance across multiple NPCs being updated on different threads would result in severe race conditions and unpredictable AI behavior.

    **WARNING:** The design mandates that each active NPC has its own distinct instance of its behavior tree, including all Sensor nodes. These components must never be shared between active AI agents.

## API Surface
The public contract is minimal, focusing exclusively on the evaluation logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(N) | Evaluates each of the N child Sensors in its internal list. Returns true on the first successful match. Has side effects of populating internal info providers and potentially clearing the Role's marked entity. |

## Integration Patterns

### Standard Usage
Developers do not typically interact with this class directly in Java. Its use is declarative, defined within an NPC's asset files. The system then uses the resulting object during the AI update loop. The following example simulates how the AI system might invoke the sensor.

```java
// Hypothetical AI update loop for a single NPC
void processAiTick(Ref<EntityStore> entityRef, Role npcRole, double deltaTime, Store<EntityStore> worldStore) {
    // The 'fleeCondition' would be a SensorOr instance loaded from an asset
    Sensor fleeCondition = npcRole.getBehaviorTree().getSensor("fleeCondition");

    // The system evaluates the sensor. If true, the NPC might change state.
    if (fleeCondition.matches(entityRef, npcRole, deltaTime, worldStore)) {
        npcRole.getStateMachine().transitionTo("Fleeing");
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SensorOr(...)` in gameplay logic. This bypasses the asset pipeline and the required builder infrastructure, leading to improperly configured and non-functional AI components. All Sensors must be defined in asset files.
-   **Instance Sharing:** Do not retrieve a Sensor from one NPC's Role and use it for another. This will cause state corruption and race conditions due to the non-thread-safe internal state.
-   **Post-Construction Modification:** The internal list of child sensors is not designed to be modified after the object is constructed by the asset builder. Attempting to do so via reflection or other means will lead to undefined behavior.

## Data Pipeline
SensorOr acts as a processing and aggregation node for world-state data. It transforms multiple boolean checks into a single boolean result, while also capturing context from the first successful check.

> Flow:
> NPC AI Tick -> Behavior Tree Evaluation -> **SensorOr.matches()** -> Iterates Child `Sensor.matches()` -> Aggregates Boolean Results -> Single `true`/`false` Output -> AI State Transition

