---
description: Architectural reference for BuilderBodyMotionBase
---

# BuilderBodyMotionBase

**Package:** com.hypixel.hytale.server.npc.corecomponents.builders
**Type:** Utility

## Definition
```java
// Signature
public abstract class BuilderBodyMotionBase extends BuilderMotionBase<BodyMotion> {
```

## Architecture & Concepts
BuilderBodyMotionBase is an abstract foundational class within the server-side NPC instruction system. It does not represent a concrete behavior but rather establishes a strict contract for a family of builders responsible for creating *body-specific* motions.

Architecturally, this class employs specialization. It extends the generic BuilderMotionBase and permanently binds it to the BodyMotion instruction category. This is achieved by providing a final implementation of the `category` method. By handling this responsibility at the base level, all concrete subclasses (e.g., a hypothetical BuilderBodyLookAt or BuilderBodyTurnTo) are simplified and guaranteed to produce the correct type of instruction.

This class is a key component of a hierarchical, fluent builder framework designed for constructing complex NPC behavior sequences in a type-safe and readable manner. It ensures that any instruction related to the NPC's main body is correctly categorized and handled by the motion execution engine.

### Lifecycle & Ownership
- **Creation:** This class is abstract and is never instantiated directly. Concrete subclasses are instantiated on-demand by higher-level NPC controllers, typically when an AI behavior tree or script requests a new body motion.
- **Scope:** The lifetime of a subclass instance is extremely short and transient. It exists only for the duration of building a single BodyMotion instruction.
- **Destruction:** The object is eligible for garbage collection immediately after the final `build` method is invoked and the resulting BodyMotion instruction is passed to the NPC's motion queue. There is no persistent state and no manual cleanup is required.

## Internal State & Concurrency
- **State:** This abstract base class is stateless. However, its concrete subclasses are inherently stateful and mutable. They accumulate parameters (e.g., target location, duration, rotation) through a series of method calls before the final instruction is built.
- **Thread Safety:** Implementations are **not thread-safe**. A builder instance is designed to be confined to the thread that created it, which is typically the primary server tick thread governing the NPC's logic. Sharing a builder instance across multiple threads will result in race conditions and corrupted instruction data.

## API Surface
The public contract of this specific class is minimal, as its primary purpose is to enforce a type contract on its children.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| category() | Class<BodyMotion> | O(1) | Returns the class literal for BodyMotion. This is a final method that cannot be overridden, guaranteeing all subclasses belong to the body motion category. |

## Integration Patterns

### Standard Usage
This class is not used directly. A developer interacts with one of its concrete subclasses through a fluent API, typically exposed by an NPC's central motion controller.

```java
// A hypothetical concrete builder (e.g., BodyLookAtBuilder) would be used like this.
// The builder itself extends BuilderBodyMotionBase.

// Obtain the NPC's motion controller
MotionController controller = npc.getMotionController();

// Chain calls to configure and build the instruction
BodyMotion lookAtPlayer = controller.body()
    .lookAt(somePlayer.getPosition())
    .withDuration(5.0f)
    .build();

// Enqueue the resulting instruction for execution
controller.enqueue(lookAtPlayer);
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** The compiler will prevent direct instantiation via `new BuilderBodyMotionBase()`. This is by design.
- **Builder Reuse:** Do not hold a reference to a concrete builder and reuse it after calling `build()`. The internal state of the builder is not guaranteed to be reset and may produce invalid subsequent instructions. Always start a new builder chain for each new motion.

## Data Pipeline
This class is a data *creator*, not a data processor. It sits at the beginning of the NPC motion pipeline, translating high-level commands into concrete instruction objects.

> Flow:
> AI Behavior Tree or Scripting Engine -> **Concrete Subclass of BuilderBodyMotionBase** -> `build()` Invocation -> New `BodyMotion` Object -> NPC Motion Queue -> Motion Executor Engine

