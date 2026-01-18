---
description: Architectural reference for BuilderExpressionDynamicBooleanArray
---

# BuilderExpressionDynamicBooleanArray

**Package:** com.hypixel.hytale.server.npc.asset.builder.expression
**Type:** Transient

## Definition
```java
// Signature
public class BuilderExpressionDynamicBooleanArray extends BuilderExpressionDynamic {
```

## Architecture & Concepts
The BuilderExpressionDynamicBooleanArray is a concrete implementation of a compiled, executable expression within the Hytale NPC Expression Language engine. It represents a script or formula that resolves to a `boolean[]` at runtime. This class is a terminal product of the expression parsing and compilation pipeline.

Its primary role is to encapsulate a pre-compiled sequence of instructions that, when executed, produce a dynamic result based on runtime game state. This allows game designers to define complex, state-dependent logic in asset files without modifying core engine code. For example, an expression could determine which of an entity's limbs are currently affected by a status effect, returning an array of boolean flags.

This class is a critical component of the data-driven NPC system. It acts as a bridge between static asset data and the dynamic, mutable state of the game world, providing a performant mechanism for evaluating logic on-demand. The evaluation itself is handled by a stack-based virtual machine, where this class provides the "bytecode" (the instruction sequence) and the caller provides the runtime environment (the ExecutionContext).

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the internal expression compiler during the asset loading phase. The compiler parses a string expression from an asset file and transforms it into an `Instruction[]` array, which is then passed to this class's constructor.
- **Scope:** The object's lifetime is bound to the parent asset that defines it, such as an NPC definition or a behavior tree. It persists in memory as long as the parent asset is loaded.
- **Destruction:** The object is marked for garbage collection when its parent asset is unloaded from memory. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** This object is effectively **immutable**. Its internal state, consisting of the original expression string and the compiled `instructionSequence`, is established at construction and is not modified thereafter. The dynamic nature of the class refers to the *result* of its execution, not the state of the object itself.

- **Thread Safety:** The BuilderExpressionDynamicBooleanArray instance is inherently thread-safe due to its immutability. However, its evaluation methods are **not reentrant or thread-safe** in the context of their arguments. The `ExecutionContext` parameter is a mutable state container for the virtual machine.

    **WARNING:** Callers are responsible for ensuring that a single `ExecutionContext` instance is not mutated by multiple threads simultaneously. A common and required pattern is to use a thread-local or single-use `ExecutionContext` for each evaluation.

## API Surface
The public API is minimal, focusing solely on type identification and expression evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | ValueType | O(1) | Returns the static type discriminator, `ValueType.BOOLEAN_ARRAY`. |
| getBooleanArray(executionContext) | boolean[] | O(N) | Executes the instruction sequence against the context. Returns the resulting boolean array from the context's operand stack. Throws if the stack is in an invalid state. |
| updateScope(scope, name, context) | void | O(N) | Executes the expression and assigns the resulting boolean array to a named variable within the provided `StdScope`. This is a side-effecting operation on the scope. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most game logic systems. It is typically invoked by higher-level systems like behavior trees or animation state machines that manage the `ExecutionContext`.

```java
// Assume 'npcAsset' holds a compiled expression of this type
BuilderExpressionDynamicBooleanArray limbStatusExpr = npcAsset.getLimbStatusExpression();

// Obtain a context for the specific NPC being evaluated
ExecutionContext npcContext = npc.getExpressionContext();

// Evaluate the expression to get the current limb status
boolean[] activeLimbs = limbStatusExpr.getBooleanArray(npcContext);

// Use the result in game logic
animationSystem.updateLimbEffects(npc, activeLimbs);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderExpressionDynamicBooleanArray()`. These objects are exclusively the output of the expression compilation service. Manually creating one would result in a disconnected and non-functional object.
- **Stateful Execution:** Do not attempt to reuse an `ExecutionContext` across evaluations of different expressions or for different entities without resetting it. The context's stack is stateful and will be corrupted, leading to unpredictable behavior and runtime exceptions.

## Data Pipeline
The creation and use of this class is a multi-stage process that transforms designer-authored text into a runtime value.

> Flow:
> NPC Asset File (e.g., JSON) -> Expression Parser -> **BuilderExpressionDynamicBooleanArray** (with compiled instructions) -> Behavior Tree Node -> `getBooleanArray(context)` -> `boolean[]` -> Game Logic (e.g., Animation System)

