---
description: Architectural reference for MotionSequence
---

# MotionSequence

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Stateful Component

## Definition
```java
// Signature
public abstract class MotionSequence<T extends Motion> extends MotionBase implements IAnnotatedComponentCollection {
```

## Architecture & Concepts

The MotionSequence is a foundational component in the NPC behavior system, implementing the **Composite** design pattern. It functions as a specialized `Motion` that, instead of defining a singular movement behavior, orchestrates an ordered sequence of child `Motion` objects. This allows for the construction of complex, multi-stage NPC actions from simple, reusable motion primitives.

Architecturally, MotionSequence acts as a state machine and a sequential task runner. Its primary role is to manage the lifecycle of its child motions, advancing from one step to the next only after the current step has completed. It abstracts the complexity of a multi-part behavior, presenting it to the higher-level `Role` or behavior tree as a single, cohesive `Motion`.

For example, a behavior like "patrol to a point, wait, and then look around" can be constructed by composing three distinct `Motion` objects (e.g., `MoveTo`, `Wait`, `LookRandomly`) within a single MotionSequence. The NPC's `Role` simply activates this sequence, and the sequence handles the transitions internally.

## Lifecycle & Ownership

-   **Creation:** A MotionSequence is never instantiated directly. It is constructed via a corresponding `BuilderMotionSequence` implementation. This builder is typically configured and invoked within an NPC's `Role` definition or a behavior tree during the NPC's initialization phase. Each instance is unique to a specific, high-level NPC objective.

-   **Scope:** The lifetime of a MotionSequence is tightly coupled to the NPC's current task. It persists only as long as the NPC is executing that specific sequence of actions. When the NPC's AI decides to switch to a different high-level behavior (e.g., from "patrol" to "flee"), the existing MotionSequence is discarded.

-   **Destruction:** The component is cleaned up by the Java garbage collector. There is no explicit `destroy` or `dispose` method. It becomes eligible for collection as soon as the `Role` or other owning component releases its reference.

## Internal State & Concurrency

-   **State:** This component is highly stateful and mutable. It internally tracks the `index` of the current step, a boolean `finished` flag, and a reference to the `activeMotion`. While its configuration (the `steps` array, `looped` flag) is immutable after construction, its execution progress is constantly changing.

-   **Thread Safety:** **WARNING:** This class is not thread-safe and is designed for single-threaded access. All state-mutating methods, particularly `computeSteering` and `activate`, must be called exclusively from the server's main game loop thread. Unsynchronized access from other threads will result in state corruption, race conditions, and severe, unpredictable NPC behavior.

## API Surface

The public contract is focused on lifecycle management and steering computation, consistent with the `Motion` interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| activate(...) | void | O(1) | Begins execution of the sequence. If `restartOnActivate` is true, it resets to the first step. |
| deactivate(...) | void | O(1) | Immediately halts the sequence and deactivates the currently running child motion. |
| computeSteering(...) | boolean | O(k) | The core update method. Delegates steering computation to the active child motion. Advances the sequence if the child completes. *k* is the number of child motions that can complete within a single tick. |
| restart() | void | O(1) | Resets the sequence's internal state to its first step, allowing it to be run again. |

## Integration Patterns

### Standard Usage

A MotionSequence is defined and used as a single `Motion` within a higher-level component like a `Role`. The builder pattern is the only supported method for creation.

```java
// Within an NPC Role or Behavior definition
// Assume 'builder' is a concrete BuilderMotionSequence instance

Motion patrolAndLook = builder
    .add(new MoveTo(destinationPoint))
    .add(new Wait(Duration.ofSeconds(5)))
    .add(new LookRandomly(Duration.ofSeconds(3)))
    .setLooped(true)
    .build();

// The Role would later activate this composite motion
this.setActiveMotion(patrolAndLook);
```

### Anti-Patterns (Do NOT do this)

-   **State Manipulation:** Do not attempt to modify the internal state of a MotionSequence externally. Calling `restart` is the only supported way to reset its progress. Manually changing the `index` or `activeMotion` will break the state machine.

-   **Reusing Builders:** A builder instance should not be reused after its `build` method has been called if its internal state is mutable. Always create a new builder for a new sequence definition.

-   **Empty Sequences:** Activating a sequence with zero steps will throw an `IllegalArgumentException`. Always ensure a sequence has at least one child motion.

## Data Pipeline

The MotionSequence acts as a controller in a flow, not a data transformer. Its primary job is to select which child `Motion` gets to influence the NPC's steering output on a given tick.

> Flow:
> NPC Role / Behavior Tree -> `activate(MotionSequence)` -> Server Game Tick -> **MotionSequence.computeSteering()** -> `activeMotion.computeSteering()` -> Steering Output -> MotionController

