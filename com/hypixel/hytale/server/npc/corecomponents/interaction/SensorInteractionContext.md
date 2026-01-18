---
description: Architectural reference for SensorInteractionContext
---

# SensorInteractionContext

**Package:** com.hypixel.hytale.server.npc.corecomponents.interaction
**Type:** Component / Transient

## Definition
```java
// Signature
public class SensorInteractionContext extends SensorBase {
```

## Architecture & Concepts
The SensorInteractionContext is a specialized predicate component within the server-side NPC AI framework. It functions as a conditional check, or a "sensor," that allows an NPC's behavior tree to make decisions based on the state of its current interaction target.

Its primary role is to answer a single question: "Does my current target meet a specific, named condition?" This condition is defined by a simple string, the *interactionContext*. For example, a context could be "isHoldingQuestItem" or "isAggressive".

This class acts as a bridge between the abstract AI logic (the behavior tree) and the concrete game state. It queries the NPC's current `Role` to identify the interaction target and then delegates the actual context validation to the `StateSupport` system. This design decouples the sensor from the implementation details of how interaction contexts are defined and checked, promoting a clean separation of concerns.

## Lifecycle & Ownership
- **Creation:** This object is not instantiated directly via its constructor. It is created by the NPC asset loading system through a corresponding builder, `BuilderSensorInteractionContext`. This occurs when an NPC's definition is loaded from asset files into memory, typically at server startup or when new zone data is loaded.
- **Scope:** The lifecycle of a SensorInteractionContext instance is tightly bound to the lifecycle of the NPC entity that owns it. It is created once when the NPC is defined and persists for the entire duration of the NPC's existence in the world.
- **Destruction:** The object is eligible for garbage collection when its parent NPC entity is unloaded or destroyed. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** Immutable. The core state of this component is the `interactionContext` string, which is marked as `final` and set only once during construction. The object itself does not maintain any other mutable state between calls.
- **Thread Safety:** Conditionally thread-safe. While the internal state is immutable, the `matches` method operates on external, mutable game state objects like `Role` and `Store<EntityStore>`. It is designed to be called exclusively from the main server entity update thread. Invoking its methods from other threads without external synchronization on the world state will lead to race conditions and is strictly forbidden.

## API Surface
The public contract is minimal, focused entirely on the evaluation logic inherited from `SensorBase`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates the sensor's condition. Returns true if the NPC has a valid, living interaction target that satisfies the configured `interactionContext`. |
| getSensorInfo() | InfoProvider | O(1) | Intended to provide debug information. This implementation currently returns null. |

## Integration Patterns

### Standard Usage
This component is not intended for direct invocation by gameplay programmers. It is configured declaratively within an NPC's asset files. The server's AI engine then automatically calls the `matches` method during the NPC's update cycle to drive behavior.

A conceptual representation of its use within a behavior tree node might look like this:

```java
// PSEUDOCODE: How the AI system uses the sensor
boolean canInitiateTrade = npc.evaluateSensor(SensorInteractionContext.class, "canTrade");

if (canInitiateTrade) {
    npc.getBehaviorController().startBehavior(TradeBehavior.class);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SensorInteractionContext()`. The object must be constructed by the asset pipeline via its builder to ensure the `interactionContext` is correctly resolved from asset data.
- **External State Management:** Do not cache the result of the `matches` method. The state of the interaction target can change on any tick, and the sensor must be re-evaluated each time a decision is required.
- **Asynchronous Execution:** Do not call the `matches` method from a separate thread. All interactions with this sensor must occur on the main server thread that owns the game world state.

## Data Pipeline
This component acts as a gate in a control flow rather than a step in a data processing pipeline. It consumes game state and produces a boolean value that influences the NPC's subsequent actions.

> Flow:
> NPC Behavior Tree Tick -> **SensorInteractionContext.matches()** -> [Boolean Result] -> Behavior Tree State Transition (e.g., Idle to Follow)

