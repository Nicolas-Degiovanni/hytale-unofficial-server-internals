---
description: Architectural reference for HeadMotionBase
---

# HeadMotionBase

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Component Base

## Definition
```java
// Signature
public abstract class HeadMotionBase extends MotionBase implements HeadMotion {
```

## Architecture & Concepts
HeadMotionBase is an abstract base class that serves as the foundational component for all NPC head-related behaviors within the server's AI system. It is a specialization of the more generic MotionBase, inheriting common lifecycle and state management logic for time-based actions.

This class establishes a common structure and shared logic for any instruction that manipulates an NPC's head orientation, such as looking at a specific point, tracking an entity, or performing a canned animation. It implements the HeadMotion interface, which guarantees that all concrete subclasses adhere to a standardized contract expected by the NPC behavior scheduler.

It is not intended for direct use. Instead, developers must extend this class to create specific, concrete head motions (e.g., LookAtPointMotion, TrackEntityMotion). The system relies on the Builder pattern, enforced by its constructor, for the safe and consistent creation of these motion instances.

## Lifecycle & Ownership
- **Creation:** Never instantiated directly due to its abstract nature. Concrete subclasses are created via their corresponding Builder (e.g., BuilderHeadMotionBase). These builders are typically invoked by higher-level AI systems, such as a Behavior Tree or a State Machine, in response to a specific stimulus or scripted event.
- **Scope:** The object's lifetime is ephemeral and tied directly to the execution of a single NPC action. It is a transient, command-like object that exists only for the duration of the specific head movement it represents.
- **Destruction:** Once the motion is complete (as indicated by a method like isFinished), the NPC's behavior system discards its reference to the instance. The object is then reclaimed by the Java Garbage Collector. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** While HeadMotionBase itself may contain minimal state, its concrete subclasses are inherently stateful. They hold mutable data defining the motion's goal and progress, such as target coordinates, tracking entity ID, duration, and current rotational values. This state is unique to one NPC performing one action.
- **Thread Safety:** **This class is not thread-safe.** Instances are designed to be owned and operated on by a single thread, typically the main server thread or a dedicated AI update thread responsible for the parent NPC. Concurrent modification from multiple threads will lead to race conditions and undefined NPC behavior. All interactions must be synchronized with the NPC's update tick.

## API Surface
The public API is primarily defined by the contracts of its parent class, MotionBase, and the HeadMotion interface. Concrete implementations will inherit this surface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(float deltaTime) | void | O(1) | (Inherited) Advances the motion's internal state. This is the primary tick method called by the NPC's behavior processor. |
| isFinished() | boolean | O(1) | (Inherited) Returns true if the motion has completed its objective and can be discarded. |

## Integration Patterns

### Standard Usage
A developer never interacts with HeadMotionBase directly. Instead, they use a concrete implementation's builder to create an instruction, which is then submitted to an NPC's task scheduler or behavior controller.

```java
// Example assumes a concrete 'LookAtPointMotion' subclass exists
LookAtPointMotion.Builder builder = new LookAtPointMotion.Builder();
builder.setTarget(new Vector3(10, 20, 30));
builder.setDuration(5.0f);

// The NPC's AI system would then enqueue this motion
npc.getBehaviorController().enqueueMotion(builder.build());
```

### Anti-Patterns (Do NOT do this)
- **State Sharing:** Never share a single motion instance between multiple NPCs. Each instance contains state specific to one NPC's action. Reusing an instance will cause unpredictable head movements and state corruption.
- **External State Modification:** Do not modify the internal state of a motion object after it has been submitted to an NPC. All configuration must be performed via its Builder before creation.
- **Reference Hoarding:** Do not maintain a long-lived reference to a motion instance after it has completed. This will prevent garbage collection and result in a memory leak.

## Data Pipeline
HeadMotionBase sits within the NPC AI's instruction execution pipeline. It translates a high-level behavioral goal into low-level entity state changes over a series of server ticks.

> Flow:
> AI Behavior Tree Node -> **BuilderHeadMotionBase** (Configuration) -> Concrete **HeadMotionBase** Instance -> NPC Task Queue -> NPC Update Tick -> Entity Head Rotation Update

