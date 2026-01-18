---
description: Architectural reference for CircleOffsetFromArgOperation
---

# CircleOffsetFromArgOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol.loops
**Type:** Transient State Machine

## Definition
```java
// Signature
public class CircleOffsetFromArgOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The CircleOffsetFromArgOperation is a specialized flow-control instruction within the Scripted Brush system. It functions as a stateful `for` loop that iterates a set number of times, modifying the brush's spatial offset at the beginning of each iteration to trace a circular path.

Its primary architectural role is to enable the creation of complex, procedural, and circular patterns without requiring developers to manually define each operation in a sequence. It bridges declarative configuration with dynamic, in-game parameters by reading the loop's bounds (number of points, radius) directly from arguments stored on the player's active Builder Tool item.

This operation combines three distinct responsibilities:
1.  **Configuration Loading:** On its first execution, it queries the player's entity components to find the active builder tool and reads named arguments to configure the loop.
2.  **State Management:** It maintains an internal state machine, principally managed by the `repetitionsRemaining` counter, to track the loop's progress.
3.  **Execution Control:** It directly manipulates the `BrushConfig` by altering its origin offset and instructs the `BrushConfigCommandExecutor` to perform a non-sequential jump to a previously defined instruction index, effectively creating a loop.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the Hytale `Codec` framework during the deserialization of a Scripted Brush asset file. The static `CODEC` field defines the mapping between the asset configuration and the class fields. It is never created manually with `new`.
-   **Scope:** The object's lifetime is bound to its parent `BrushConfig`. Its internal state, however, is transient and relevant only to a single, continuous application of the brush. The `resetInternalState` method is invoked for each new brush stroke.
-   **Destruction:** The object is eligible for garbage collection when its parent `BrushConfig` is unloaded, typically when server assets are reloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This class is highly mutable and stateful. It caches configuration values read from the player's tool (`numCirclePointsArgVal`, `circleRadiusArgVal`) and maintains a loop counter (`repetitionsRemaining`). Crucially, it pre-calculates and caches the list of all offset vectors (`offsetsInCircle`) upon first execution to avoid expensive trigonometric calculations on every iteration. It also stores the initial brush offset (`offsetWhenFirstReachedOperation`) to restore the context after the loop terminates.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be operated exclusively by the `BrushConfigCommandExecutor` on the main server thread. All state modifications are performed sequentially within a single game tick. Concurrent access would corrupt the loop counter and offset calculations, leading to undefined brush behavior.

## API Surface
The public contract is exclusively designed for interaction with the Scripted Brush execution engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(N) / O(1) | Executes the operation. The first call has O(N) complexity to pre-calculate N circle points. Subsequent calls are O(1). Manages the loop state machine, updates the BrushConfig origin offset, and directs the command executor to jump. |
| resetInternalState() | void | O(N) | Resets the loop counter and recalculates the cached list of offset vectors based on the current configuration. This is considered part of the internal lifecycle and should not be called externally. |

## Integration Patterns

### Standard Usage
This operation is not invoked directly via Java code. It is defined declaratively within a Scripted Brush asset file (e.g., a JSON or HOCON file). The engine's `Codec` system handles its instantiation and integration into the operation sequence.

A conceptual configuration might look like this:
```yaml
# In a conceptual brush asset definition
name: "Cylinder Brush"
sequence: [
  {
    # ... other operations
  },
  {
    # Store the current instruction index with the name "LoopStart"
    type: "hytale:store_index"
    name: "LoopStart"
  },
  {
    # ... operations to be repeated, e.g., place a block
  },
  {
    # The loop controller
    type: "hytale:circle_offset_from_arg"
    StoredIndexName: "LoopStart"
    NumberCirclePointsArg: "cylinder_points" # Reads from a tool arg named "cylinder_points"
    CircleRadiusArg: "cylinder_radius"     # Reads from a tool arg named "cylinder_radius"
    FlipDirection: false
    RotateDirection: false
  }
]
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new CircleOffsetFromArgOperation()`. The object will be unconfigured and will cause `NullPointerException` or other errors when executed. Always define it within a brush asset file.
-   **External State Modification:** Do not attempt to get an instance of this operation and modify its state (e.g., `repetitionsRemaining`) externally. The internal logic is a delicate state machine that relies on its own managed transitions.
-   **Incorrect Indexing:** The `StoredIndexName` must refer to an index that appears *before* this operation in the sequence. Referring to a future index will result in an infinite loop or undefined behavior.

## Data Pipeline
This operation acts as a control node that both reads from and writes to the central `BrushConfig` object, while also directing the flow of the `BrushConfigCommandExecutor`.

> Flow:
> 1.  **Player Tool Arguments** (e.g., radius) -> `modifyBrushConfig` (Initialization on first run)
> 2.  **Internal State** (Pre-calculated `offsetsInCircle` list is populated)
> 3.  **BrushConfig** (Current `originOffset`) -> `modifyBrushConfig` (Calculation)
> 4.  `modifyBrushConfig` -> **BrushConfig** (Writes new `originOffset`)
> 5.  `modifyBrushConfig` -> **BrushConfigCommandExecutor** (Issues a `loadOperatingIndex` command to jump)
> 6.  (Loop repeats until `repetitionsRemaining` is zero)
> 7.  `modifyBrushConfig` -> **BrushConfig** (Restores original `originOffset`)

