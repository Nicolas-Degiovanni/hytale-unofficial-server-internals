---
description: Architectural reference for BuilderExpressionDynamicNumberArray
---

# BuilderExpressionDynamicNumberArray

**Package:** com.hypixel.hytale.server.npc.asset.builder.expression
**Type:** Transient

## Definition
```java
// Signature
public class BuilderExpressionDynamicNumberArray extends BuilderExpressionDynamic {
```

## Architecture & Concepts
The BuilderExpressionDynamicNumberArray is a specialized, executable representation of a server-side NPC expression. It encapsulates a pre-compiled sequence of instructions that, when evaluated, produces a dynamic array of floating-point numbers. This class is a fundamental component of the server's expression engine, which allows content creators to define complex NPC behaviors and attributes using a simple scripting language.

Architecturally, this class acts as a *Compiled Expression Object*. It is the final output of the expression parsing and compilation pipeline. It holds an immutable instruction set, separating the static definition of the logic from its runtime execution. The actual evaluation is delegated to a stateful ExecutionContext, making this class a stateless and reusable component.

This design follows the **Interpreter Pattern**, where this class is a terminal expression that knows how to evaluate itself within a given context.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the internal expression compiler when it parses a script that is determined to resolve to a numeric array. It is never instantiated directly by game logic. The constructor receives the raw expression string for debugging and the compiled instruction sequence for execution.
- **Scope:** The lifetime of a BuilderExpressionDynamicNumberArray instance is bound to the lifecycle of the NPC asset that defines it. It persists in memory as long as the corresponding NPC definition is loaded.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources or file handles and is safely collected once the parent NPC asset is unloaded and no longer referenced.

## Internal State & Concurrency
- **State:** This class is **immutable**. Its internal fields, the expression string and the instruction sequence, are set at construction and are not modified thereafter. All state required for evaluation, such as the operand stack and variable values, is stored externally in the ExecutionContext object passed into its methods.

- **Thread Safety:** The BuilderExpressionDynamicNumberArray object itself is inherently **thread-safe** due to its immutable nature. Multiple threads can invoke methods on a shared instance without risk of data corruption.

    **WARNING:** Thread safety of an *evaluation* depends entirely on the provided ExecutionContext. The ExecutionContext is stateful and is **not** thread-safe. Executing the same expression from multiple threads requires each thread to use its own distinct ExecutionContext instance to prevent severe race conditions on the internal operand stack.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | ValueType | O(1) | Returns the static type identifier, ValueType.NUMBER_ARRAY. |
| getNumberArray(executionContext) | double[] | O(N) | Executes the instruction sequence against the context and returns the resulting double array from the context's stack. N is the number of instructions. |
| getIntegerArray(executionContext) | int[] | O(N+M) | Executes the expression and converts the resulting double array to an integer array. M is the length of the resulting array. |
| updateScope(scope, name, context) | void | O(N) | Executes the expression and assigns the resulting array to a named variable within the provided StdScope. |

## Integration Patterns

### Standard Usage
This object is typically retrieved from a parent asset and executed to configure an NPC's state. The caller is responsible for providing a correctly initialized execution context.

```java
// Assume 'npcBehavior' is an object holding the compiled expressions
// Assume 'executionContext' is prepared for this specific NPC instance

// Retrieve the compiled expression for a property, e.g., "patrolPathOffsets"
BuilderExpressionDynamicNumberArray pathOffsetsExpr = npcBehavior.getExpression("patrolPathOffsets");

// Execute the expression to get the runtime value
double[] offsets = pathOffsetsExpr.getNumberArray(executionContext);

// Use the result in the game logic
npc.getMovementController().setPathOffsets(offsets);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BuilderExpressionDynamicNumberArray()`. These objects are the output of the asset compilation pipeline. Constructing them manually bypasses this system and will result in a non-functional object.
- **Shared ExecutionContext:** Do not share a single ExecutionContext instance across multiple threads to evaluate expressions. This will corrupt the operand stack and lead to unpredictable behavior and crashes.
- **Incorrect Type Assumption:** Do not blindly cast a generic BuilderExpression to this type. Always check the result of `getType()` before attempting to call `getNumberArray` to avoid runtime ClassCastExceptions.

## Data Pipeline
The data flow for creating and using this class involves both asset loading (compile-time) and game logic (run-time).

> **Compile-Time Flow:**
> NPC Definition File (e.g., JSON) -> Expression String Parser -> Tokenizer -> Instruction Compiler -> **new BuilderExpressionDynamicNumberArray(instructions)** -> Stored in NPC Asset

> **Run-Time Flow:**
> Game Logic -> Request Expression Result -> **getNumberArray(ExecutionContext)** -> Internal `execute(context)` -> `double[]` Result -> NPC System (e.g., Pathfinding, AI)

