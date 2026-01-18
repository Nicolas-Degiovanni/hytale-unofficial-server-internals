---
description: Architectural reference for CircleOffsetAndLoopOperation
---

# CircleOffsetAndLoopOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol.loops
**Type:** Transient

## Definition
```java
// Signature
public class CircleOffsetAndLoopOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The **CircleOffsetAndLoopOperation** is a specialized flow-control instruction within the Scripted Brush system. It is not an operation that directly modifies the world; instead, it functions as a stateful loop controller that manipulates the execution context of a brush script.

Its primary architectural role is to enable the repetition of a sequence of prior operations in a circular pattern. It achieves this by performing two critical actions during each iteration:

1.  **Execution Pointer Manipulation:** It instructs the **BrushConfigCommandExecutor** to jump back to a previously defined, named index in the operation sequence, effectively creating a loop.
2.  **State Modification:** It systematically modifies the **originOffset** within the active **BrushConfig**. The offset is moved to the next pre-calculated point on a virtual circle, causing subsequent world-modifying operations (like placing blocks) to execute at a new position relative to the brush's initial origin.

This class acts as a state machine, driven by the **BrushConfigCommandExecutor**. Its internal state, primarily the **repetitionsRemaining** counter, determines whether to continue the loop, exit the loop, or perform first-time initialization.

## Lifecycle & Ownership
-   **Creation:** An instance of **CircleOffsetAndLoopOperation** is never created directly with the *new* keyword in game logic. It is instantiated by the engine's **Codec** system when a Scripted Brush definition is parsed from its configuration file. The static **CODEC** field defines how to deserialize the configuration properties into the class instance's fields.

-   **Scope:** The object's lifetime is strictly bound to the parent **BrushConfig** that contains it. It persists as a configured operation within the brush's script as long as that brush is loaded.

-   **Destruction:** The instance is eligible for garbage collection when its parent **BrushConfig** is unloaded. This typically occurs when a player selects a different brush, leaves the server, or the brush definitions are reloaded. The internal state is reset via **resetInternalState** at the beginning of each full brush application, but the object itself persists.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains several fields to manage the loop's execution:
    -   **repetitionsRemaining:** A counter that tracks the number of loop iterations left. It is initialized to -1 (**IDLE_STATE**) and is the primary driver of the internal state machine.
    -   **offsetsInCircle:** A cached list of **Vector3i** objects representing the pre-calculated points on the circle. This cache is populated once upon the first execution of the operation to avoid redundant trigonometric calculations on subsequent iterations.
    -   **offsetWhenFirstReachedOperation:** A snapshot of the brush's origin offset before the loop began. This is crucial for restoring the brush's state after the loop completes.
    -   **previousCircleOffset:** Tracks the last offset applied to correctly calculate the next relative position.

-   **Thread Safety:** **This class is not thread-safe and must not be accessed from multiple threads.** It is designed to be executed exclusively by the main server thread as part of the sequential processing of a **BrushConfigCommandExecutor**. Its methods directly mutate its own state and the state of the passed-in **BrushConfig** without any synchronization mechanisms. This is by design, as builder tool operations are processed in a strictly linear and single-threaded fashion.

## API Surface
The primary contract is with the **BrushConfigCommandExecutor** via the **modifyBrushConfig** method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(ref, brushConfig, executor, accessor) | void | O(N) / O(1) | The core execution entry point. On first call, it performs an O(N) calculation to generate circle points. Subsequent calls are O(1). Modifies the **BrushConfig** offset and directs the **executor** to jump to a new instruction index. |
| resetInternalState() | void | O(N) | Overrides the parent method. Resets all internal counters and recalculates the list of circle offsets. N is **numberOfCirclePointsArg**. |

## Integration Patterns

### Standard Usage
This operation is not invoked directly from code. It is defined declaratively within a Scripted Brush configuration file. The user defines a sequence of operations, marks a starting point with a **StoreIndexOperation**, and then uses **CircleOffsetAndLoopOperation** to loop back to that point.

A conceptual brush definition might look like this:

```yaml
# This is a conceptual representation, not actual file syntax.
name: "Circular Pillar Brush"
operations:
  - type: StoreIndexOperation
    name: "LoopStart"
  - type: PlaceBlockOperation
    block: "hytale:stone"
  - type: MoveOffsetOperation
    offset: [0, 1, 0] # Move up one block
  - type: CircleOffsetAndLoopOperation
    StoredIndexName: "LoopStart"
    NumberOfCirclePoints: 8
    CircleRadius: 5
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new CircleOffsetAndLoopOperation()`. The object is fundamentally dependent on being configured and managed by the **Codec** and **BrushConfig** systems.
-   **External State Modification:** Do not attempt to modify the internal state (e.g., **repetitionsRemaining**) from outside the class. The internal state machine is delicate and relies on its own managed transitions.
-   **Invalid Loop Index:** Pointing **StoredIndexName** to an index that appears *after* this operation will result in undefined behavior or an infinite loop, as the executor will jump forward and may never re-evaluate the loop condition.

## Data Pipeline
The flow of control and data is orchestrated by the **BrushConfigCommandExecutor**. This operation acts as a node in that flow, redirecting control and modifying the shared state (**BrushConfig**).

> Flow:
> **BrushConfigCommandExecutor** executes operations sequentially -> It invokes **modifyBrushConfig** on this instance -> **CircleOffsetAndLoopOperation** reads and writes **BrushConfig.originOffset** -> **CircleOffsetAndLoopOperation** calls **BrushConfigCommandExecutor.loadOperatingIndex** -> The executor's internal instruction pointer is reset -> Execution resumes at a previous operation in the sequence.

