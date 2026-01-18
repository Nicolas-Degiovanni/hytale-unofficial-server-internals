---
description: Architectural reference for JumpToRandomIndex
---

# JumpToRandomIndex

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol
**Type:** Transient Operation

## Definition
```java
// Signature
public class JumpToRandomIndex extends SequenceBrushOperation {
```

## Architecture & Concepts
The JumpToRandomIndex class is a flow-control operation within the Scripted Brushes framework. It functions as a conditional "GOTO" statement, altering the standard sequential execution of a brush's operation list.

Its primary role is to introduce probabilistic behavior into brush scripts. Instead of executing operations A, B, C in order, a script can execute A, then use JumpToRandomIndex to decide whether to jump to operation M or operation X based on a weighted probability distribution. This allows for the creation of more organic and less repetitive procedural generation tools.

This operation is stateless during execution; its behavior is defined entirely by its configuration data. It integrates directly with the BrushConfigCommandExecutor, which maintains the "program counter" or current operating index for the active brush script. By calling `loadOperatingIndex`, this class directly manipulates the executor's state to redirect the flow of the script.

### Lifecycle & Ownership
- **Creation:** Instances of JumpToRandomIndex are not created directly via their constructor in game logic. They are instantiated by the engine's codec system when a BrushConfig asset is deserialized from storage. The public static final CODEC field defines the contract for this process.
- **Scope:** The object's lifetime is bound to the parent BrushConfig that contains it. It is a value object within a larger configuration graph and persists as long as the brush is loaded in memory.
- **Destruction:** The object is marked for garbage collection when its parent BrushConfig is unloaded or replaced. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** The core state is the `variableNameArg` field, an IWeightedMap mapping string-based index names to their selection weights. This state is populated once during deserialization and is treated as immutable thereafter. The operation itself does not modify its own state during execution.
- **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent access. All interactions are expected to occur on the main game thread responsible for executing the brush logic. The system relies on the single-threaded execution model of the Scripted Brush executor to ensure correctness.

## API Surface
The primary contract is the `modifyBrushConfig` method, which is invoked by the brush execution engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(N) | Selects a target index name from its weighted map and instructs the BrushConfigCommandExecutor to jump to that index. N is the number of potential targets. |
| CODEC | BuilderCodec | N/A | A static field defining the serialization and deserialization contract for this operation. |

## Integration Patterns

### Standard Usage
This operation is not used by writing Java code, but by defining it within a brush's configuration data, typically a JSON or similar asset file. The engine's `CODEC` handles the instantiation.

The following conceptual example shows how one might define a jump operation where there is a 75% chance of jumping to the "AddGrass" index and a 25% chance of jumping to the "AddFlowers" index.

```json
// Conceptual Brush Script Definition
{
  "operation": "JumpToRandomIndex",
  "config": {
    "WeightedListOfIndexNames": [
      { "left": 75, "right": "AddGrass" },
      { "left": 25, "right": "AddFlowers" }
    ]
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new JumpToRandomIndex()`. The resulting object will be unconfigured and will perform no action. Always define operations via the appropriate data assets to ensure the `CODEC` correctly populates them.
- **Referencing Non-Existent Indices:** The index names provided in the configuration (e.g., "AddGrass") must correspond to a `StoreIndex` operation defined elsewhere in the same script. A jump to an undefined index will result in a runtime error and halt the brush execution.
- **Zero Weights:** Providing a list of targets where all weights are zero will lead to undefined behavior in the weighted map selection logic. At least one target must have a positive weight.

## Data Pipeline
The flow of control begins with the brush executor and is redirected by this operation.

> Flow:
> Brush Executor advances to next operation -> Executor invokes **JumpToRandomIndex.modifyBrushConfig** -> Operation reads from its internal WeightedMap -> Operation calls **BrushConfigCommandExecutor.loadOperatingIndex** -> Executor's internal program counter is updated -> Executor continues execution from the new index on the next step.

