---
description: Architectural reference for JumpIfCompareOperation
---

# JumpIfCompareOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.sequential.flowcontrol
**Type:** Transient

## Definition
```java
// Signature
public class JumpIfCompareOperation extends SequenceBrushOperation {
```

## Architecture & Concepts
The JumpIfCompareOperation is a fundamental flow-control instruction within the Scripted Brush engine. It functions as a conditional jump, analogous to a `GOTO` or `JUMP-IF-EQUAL` instruction in low-level programming. Its primary purpose is to alter the standard, linear execution of a brush's operation sequence.

This operation is a component within a larger state machine managed by a BrushConfigCommandExecutor. When the executor encounters this operation, it does not directly modify the game world. Instead, it evaluates a set of predefined conditions against the current state of the active BrushConfig. If all conditions are met (using a logical AND), it commands the executor to change its internal instruction pointer to a new, labeled position within the operation sequence. If any condition fails, this operation does nothing, and the executor proceeds to the next instruction in the sequence.

This mechanism enables the creation of complex, stateful, and dynamic builder tools that can loop, branch, and react to their own internal configuration without requiring custom Java code for every new behavior.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale Codec system during the deserialization of a Scripted Brush definition file (e.g., a JSON asset). The static CODEC field dictates how the configuration data maps to the object's fields. It is never instantiated directly in game logic code.
- **Scope:** The object's lifetime is bound to the loaded Scripted Brush asset. It is effectively a stateless, immutable configuration object that persists as long as the brush definition is held in memory.
- **Destruction:** The object is eligible for garbage collection when the parent Scripted Brush definition is unloaded or replaced. It manages no external resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The JumpIfCompareOperation is **effectively immutable** post-initialization. Its state, consisting of the comparison rules and the target index name, is configured once at load time by the Codec and is not designed to be mutated at runtime. The operation itself is stateless during execution; it reads from a BrushConfig but does not modify its own internal fields.
- **Thread Safety:** This class is inherently thread-safe due to its immutable design. However, the context in which it operates is not guaranteed to be. The `modifyBrushConfig` method must be invoked by a system that ensures synchronized access to the BrushConfig and BrushConfigCommandExecutor instances to prevent race conditions. The Scripted Brush engine is responsible for this synchronization.

## API Surface
The primary interaction point is the `modifyBrushConfig` method, which is part of the contract inherited from SequenceBrushOperation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modifyBrushConfig(...) | void | O(N) | Evaluates N configured comparisons. If all succeed, it invokes `loadOperatingIndex` on the provided executor to alter the execution flow. |

## Integration Patterns

### Standard Usage
This operation is not intended to be used directly from Java code. It is defined declaratively within a Scripted Brush asset file. The engine interprets this data to create the operation sequence.

The following pseudo-code illustrates how this operation would be defined in a configuration file.

```json
// Example Brush Definition Snippet
"operations": [
  {
    "type": "hytale:store_index",
    "StoredIndexName": "LoopStart"
  },
  // ... other operations that modify the brush config ...
  {
    "type": "hytale:jump_if_compare",
    "StoredIndexName": "LoopStart",
    "Comparisons": [
      {
        "DataGettingFlag": "BRUSH_SIZE",
        "IntegerComparisonOperator": "GREATER_THAN",
        "ValueToCompareTo": 5
      }
    ]
  }
]
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new JumpIfCompareOperation()`. The object's state is complex and must be configured via the engine's Codec system from a data source. Direct instantiation will result in a non-functional object.
- **Runtime State Mutation:** Do not attempt to modify the `comparisonsArg` or `indexVariableNameArg` fields after the object has been loaded. This violates the declarative design of the system and can lead to unpredictable behavior.
- **External Invocation:** The `modifyBrushConfig` method should only be called by the controlling BrushConfigCommandExecutor as part of its execution loop. Calling it from other contexts will bypass the state machine's logic.

## Data Pipeline
This component is a control-flow director, not a data-processing node. Its "pipeline" describes how it redirects the flow of execution.

> Flow:
> BrushConfigCommandExecutor retrieves operation -> Executor calls `modifyBrushConfig` -> **JumpIfCompareOperation** reads state from BrushConfig -> **JumpIfCompareOperation** evaluates conditions -> If true, calls `loadOperatingIndex` on Executor -> Executor updates its internal instruction pointer to a new operation.

