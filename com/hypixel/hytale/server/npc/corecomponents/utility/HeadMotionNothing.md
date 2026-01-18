---
description: Architectural reference for HeadMotionNothing
---

# HeadMotionNothing

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Behavioral Strategy

## Definition
```java
// Signature
public class HeadMotionNothing extends HeadMotionBase {
```

## Architecture & Concepts
The HeadMotionNothing class is a concrete implementation of the **Null Object Pattern** within the server-side NPC AI framework. Its primary architectural function is to provide a safe, non-operative behavior for an NPC's head movement system.

In the Hytale engine, an NPC's behavior is composed of multiple, swappable strategies for tasks like movement, targeting, and, in this case, head motion. The HeadMotionNothing component is assigned to an NPC when its current state or role dictates that no specific head motion logic should be applied. This prevents the need for null checks within the core AI update loop and provides a clear, explicit state for "no head movement".

It acts as a terminal node in the steering behavior calculation, guaranteeing that any prior head motion intent is nullified for the current tick. This is critical for synchronizing NPC behavior with animations or scripted sequences that require a static head position.

### Lifecycle & Ownership
- **Creation:** Instantiated by the NPC's behavior controller, typically a `Role` or `BehaviorTree`, when an NPC enters a state that requires head motion to be disabled. It is constructed via its corresponding builder, `BuilderHeadMotionBase`, not directly.
- **Scope:** The lifetime of a HeadMotionNothing instance is typically transient. It exists only as long as it is the active `HeadMotionBase` strategy for a given NPC. It is frequently created and discarded as an NPC transitions between different behaviors (e.g., from looking at a target to an idle state).
- **Destruction:** The object is eligible for garbage collection as soon as the NPC's AI controller replaces it with a different `HeadMotionBase` implementation, or when the parent NPC entity is destroyed.

## Internal State & Concurrency
- **State:** HeadMotionNothing is **stateless** and effectively immutable. It contains no instance fields that store data between calls. Its behavior is idempotent and depends solely on the arguments passed to its methods.
- **Thread Safety:** This class is inherently **thread-safe**. As a stateless object, it can be safely shared or accessed from multiple threads without risk of data corruption. However, in practice, it is almost always invoked exclusively from the main server thread responsible for the owning NPC's update tick. The `Steering` object it modifies is an output parameter and is not expected to be accessed concurrently.

## API Surface
The public contract is minimal, consisting only of the behavior defined by its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeSteering(...) | boolean | O(1) | Resets the provided `desiredSteering` object to a zero state. This effectively cancels any head movement for the current update tick. Always returns true, signifying a successful (though null) operation. |

## Integration Patterns

### Standard Usage
This component is not intended for direct invocation by game logic developers. Instead, it is configured declaratively as part of an NPC's behavioral definition. The AI system selects and executes it automatically.

```java
// Example: Configuring a behavior state to use HeadMotionNothing
// This is a conceptual representation of how a Role or Behavior Tree
// would be configured, not direct API usage.

npcBuilder.withState("IDLE", state ->
    state.setHeadMotion(new BuilderHeadMotionBase(HeadMotionNothing.class))
);

// The engine then internally handles the lifecycle and invocation.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new HeadMotionNothing()`. The NPC component system relies on the builder pattern (`BuilderHeadMotionBase`) for correct initialization and integration. Bypassing the builder can lead to unpredictable behavior.
- **Conditional Nulling:** Do not write code that manually nullifies a `HeadMotionBase` reference to stop head movement. Instead, assign an instance of HeadMotionNothing to explicitly and safely represent the "no operation" state.

## Data Pipeline
HeadMotionNothing acts as a sink in the head motion data pipeline, terminating the calculation for a given frame.

> Flow:
> NPC AI Tick -> Behavior State Evaluation -> **HeadMotionNothing.computeSteering()** -> Cleared Steering Object -> NPC Pose/Animation System

