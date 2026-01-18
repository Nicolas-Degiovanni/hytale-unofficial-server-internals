---
description: Architectural reference for HeadMotionTimer
---

# HeadMotionTimer

**Package:** com.hypixel.hytale.server.npc.corecomponents.timer
**Type:** Transient

## Definition
```java
// Signature
public class HeadMotionTimer extends MotionTimer<HeadMotion> implements HeadMotion {
```

## Architecture & Concepts
The HeadMotionTimer is a stateful, time-aware component responsible for executing a single, specific head motion instruction for a server-side NPC. It acts as a concrete implementation of the generic MotionTimer, specializing it for the HeadMotion type.

Architecturally, this class bridges the gap between a declarative NPC behavior asset (the what) and the imperative, tick-by-tick execution of that behavior (the how). It wraps a raw HeadMotion data object, enriching it with lifecycle management, progress tracking, and completion state.

By also implementing the HeadMotion interface, it employs a **Decorator Pattern**. This allows other systems, such as the NPC's animation or physics controllers, to interact with this timer as if it were the original, stateless HeadMotion instruction. This elegantly abstracts the complexity of time management from the systems that consume the motion's target data.

## Lifecycle & Ownership
- **Creation:** An instance of HeadMotionTimer is created exclusively by the NPC's instruction processing system via a corresponding BuilderHeadMotionTimer. This typically occurs when an NPC's Behavior Tree or State Machine selects a HeadMotion instruction to execute. The builder pattern ensures that all necessary dependencies, such as the BuilderSupport context, are correctly injected.

- **Scope:** The object's lifetime is ephemeral and is strictly bound to the duration of the head motion it controls. It exists only while the specific motion is active.

- **Destruction:** The object is not explicitly destroyed. Once the timer completes its duration, is interrupted by a higher-priority behavior, or the parent NPC is unloaded, all references to the HeadMotionTimer are dropped. It then becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** This object is highly **mutable**. Its primary function is to maintain and update internal state inherited from MotionTimer, such as elapsed time and completion status. This state represents the real-time progress of the NPC's action.

- **Thread Safety:** This class is **not thread-safe** and must not be considered as such. All interactions, particularly state-mutating calls like an update method, are expected to occur synchronously on the main server game thread during the NPC update tick. Unsynchronized access from other threads will lead to state corruption and undefined behavior.

## API Surface
The public contract is inherited from its parent MotionTimer and the HeadMotion interface. The core interaction is driven by the server's game loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(float deltaTime) | void | O(1) | Advances the timer's internal state by the given delta. This is the primary method called each server tick. |
| isFinished() | boolean | O(1) | Returns true if the motion's duration has elapsed. Used by the behavior system to transition to the next instruction. |
| (HeadMotion getters) | various | O(1) | Exposes properties of the underlying HeadMotion data, such as target pitch or yaw, by delegating the calls to the wrapped object. |

## Integration Patterns

### Standard Usage
The HeadMotionTimer is intended to be managed by the NPC's core behavior system. A controller retrieves a builder, constructs the timer for a specific motion, and updates it every tick until completion.

```java
// Pseudo-code for an NPC's update loop
HeadMotion instruction = behaviorTree.getActiveInstruction();
HeadMotionTimer activeTimer = motionTimerBuilder.build(instruction);

// In the per-tick update method:
if (!activeTimer.isFinished()) {
    activeTimer.update(server.getDeltaTime());
    npc.setHeadRotation(activeTimer.getCurrentYaw(), activeTimer.getCurrentPitch());
} else {
    // Transition to the next behavior
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new HeadMotionTimer(). The constructor requires internal context provided by the builder system. Bypassing the builder will result in a partially initialized object that will fail at runtime.
- **State Reuse:** Do not attempt to reuse a HeadMotionTimer instance after it has finished. Each new motion instruction requires a new, distinct timer instance to be built.
- **External State Mutation:** Modifying the timer's internal state from outside the class is unsupported. Only the update method should be used to advance time.

## Data Pipeline
The HeadMotionTimer is a key component in the transformation of an AI decision into a visible in-game action.

> Flow:
> AI Behavior Tree selects a HeadMotion asset -> Instruction Processor uses a **BuilderHeadMotionTimer** to create a **HeadMotionTimer** instance -> Server Tick Loop calls update() on the instance -> The timer's state is used to calculate the NPC's head rotation -> The NPC entity's rotation data is updated -> The change is replicated to clients via network packets.

