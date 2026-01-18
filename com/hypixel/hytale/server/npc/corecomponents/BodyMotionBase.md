---
description: Architectural reference for BodyMotionBase
---

# BodyMotionBase

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class BodyMotionBase extends MotionBase implements BodyMotion {
```

## Architecture & Concepts
BodyMotionBase is an abstract foundational component within the server-side NPC (Non-Player Character) motion control system. It serves as the common ancestor for all concrete instructions that dictate the physical movement and orientation of an NPC's body, such as walking, turning, or strafing.

Architecturally, this class embodies the "Instruction" part of a command pattern. It encapsulates all the necessary parameters for a single, discrete body movement. By extending MotionBase, it inherits core properties common to all NPC motions (e.g., duration, state tracking). By implementing the BodyMotion interface, it guarantees a standardized contract for the NPC's core processing loop, allowing the system to handle any type of body movement polymorphically without needing to know the specific implementation details.

This class is not intended to be used directly. Instead, developers must create concrete subclasses (e.g., WalkMotion, TurnInPlaceMotion) that define specific movement logic. The mandatory use of a Builder pattern for construction ensures that motion objects are created in a valid and immutable state.

## Lifecycle & Ownership
- **Creation:** This abstract class is never instantiated directly. Concrete subclasses are created exclusively via their corresponding BuilderBodyMotionBase implementations. These builders are typically invoked by higher-level AI constructs like Behavior Trees or State Machines when an NPC needs to perform a specific action.
- **Scope:** An instance of a BodyMotionBase subclass is highly transient. Its lifecycle is bound to the execution of a single NPC instruction. It is created, processed by the NPC's motion controller for one or more server ticks, and then discarded.
- **Destruction:** Once the motion is completed or interrupted, the object loses all references and is reclaimed by the Java garbage collector. It does not manage any native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** Instances are designed to be **effectively immutable** after construction. The Builder pattern is used to configure all state before the object is created. This design simplifies reasoning about NPC behavior, as a motion instruction cannot change its parameters midway through execution.
- **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent access. All interactions with BodyMotionBase and its subclasses must occur on the primary server thread responsible for the owning NPC's updates. Unsynchronized access from other threads will lead to unpredictable behavior and state corruption within the NPC AI.

## API Surface
The public API of any concrete BodyMotionBase implementation is primarily defined by the contracts inherited from the MotionBase parent class and the BodyMotion interface. The constructor is protected and accessible only to its builder.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BodyMotionBase(builder) | constructor | O(1) | Protected constructor. Initializes the motion from a builder. Not for public use. |
| *Inherited Methods* | *(Varies)* | *(Varies)* | The functional API is provided by the `MotionBase` and `BodyMotion` contracts. |

## Integration Patterns

### Standard Usage
Correct usage requires employing a specific builder to create a concrete motion instruction. This instruction is then typically passed to an NPC's behavior or motion controller for execution.

```java
// Example assumes a concrete 'WalkMotion' subclass and its builder exist.
WalkMotion.Builder walkBuilder = new WalkMotion.Builder(targetPosition);

// Configure the motion instruction via the builder
walkBuilder.withSpeed(MovementSpeed.RUN);
walkBuilder.facing(FacingDirection.TOWARDS_TARGET);

// Build the immutable motion object
BodyMotion walkInstruction = walkBuilder.build();

// Submit the instruction to the NPC's controller
npc.getMotionController().setBodyMotion(walkInstruction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to create a concrete subclass with `new WalkMotion()`. The internal state will be uninitialized, leading to NullPointerExceptions and invalid NPC behavior. Always use the designated Builder.
- **State Reuse:** Do not hold onto a motion instruction instance after it has been completed. These objects are transient and may not behave correctly if resubmitted to the motion controller. Generate a new instruction for each new action.

## Data Pipeline
BodyMotionBase acts as a data container in a control-flow pipeline. It translates a high-level AI decision into a low-level, executable command for the NPC's core systems.

> Flow:
> AI Behavior Tree Node -> **BuilderBodyMotionBase** (Configuration) -> **BodyMotionBase Instance** (Immutable Instruction) -> NPC Motion Controller (Execution) -> Entity Transform & Animation Updates

