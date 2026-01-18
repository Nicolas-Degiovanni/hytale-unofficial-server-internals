---
description: Architectural reference for BuilderExpressionDynamicString
---

# BuilderExpressionDynamicString

**Package:** com.hypixel.hytale.server.npc.asset.builder.expression
**Type:** Transient

## Definition
```java
// Signature
public class BuilderExpressionDynamicString extends BuilderExpressionDynamic {
```

## Architecture & Concepts
The BuilderExpressionDynamicString class is a specialized, concrete implementation within the server's expression evaluation engine. It represents a pre-compiled script or expression that, when executed, resolves to a String value.

This class embodies the final stage of a compiler-interpreter pattern. During server asset loading, human-readable string expressions (e.g., from NPC configuration files) are parsed and compiled into a low-level bytecode format, represented by an `ExecutionContext.Instruction[]` array. BuilderExpressionDynamicString acts as a lightweight, immutable container for this bytecode.

Its primary role is to decouple the expensive parsing and compilation phase (done once at load-time) from the frequent, performance-sensitive evaluation phase (done at runtime). By holding the compiled instruction sequence, it allows for rapid execution of the expression whenever required by game logic, such as determining an NPC's dialogue.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the server's internal expression compiler during the asset loading pipeline. A parser reads a string from a configuration file and produces the instruction sequence necessary for the constructor. Direct instantiation by developers is not supported or intended.
- **Scope:** The lifetime of a BuilderExpressionDynamicString instance is tied directly to its owning asset, such as an NPC definition or a behavior tree node. It persists in memory as long as the parent asset is loaded.
- **Destruction:** The object is marked for garbage collection when its parent asset is unloaded from memory, for example, when a zone or world unloads.

## Internal State & Concurrency
- **State:** This object is **effectively immutable**. The `expression` string and `instructionSequence` array are provided at construction and are never modified. The class itself is stateless; all mutable state required for evaluation (such as the operand stack) is managed externally within the provided `ExecutionContext` object.
- **Thread Safety:** The class is inherently **thread-safe**. Because its internal state is immutable, a single instance can be safely accessed and executed by multiple threads simultaneously.
    - **WARNING:** Thread safety is contingent on each thread providing its own unique, non-shared `ExecutionContext` instance. Sharing an `ExecutionContext` across threads will lead to race conditions, stack corruption, and undefined behavior.

## API Surface
The public API is focused on executing the compiled expression and retrieving or applying its result.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | ValueType | O(1) | Returns the static type of the expression, which is always `ValueType.STRING`. |
| getString(ExecutionContext) | String | O(N) | Executes the stored instruction sequence against the given context and returns the resulting String from the context's stack. N is the number of instructions. |
| updateScope(StdScope, String, ExecutionContext) | void | O(N) | Executes the expression and assigns the resulting String to a named variable within the provided `StdScope`. |

## Integration Patterns

### Standard Usage
This class is not used directly but is invoked by higher-level systems that manage NPC logic. The typical pattern involves retrieving a pre-compiled expression from an asset and evaluating it with a context built for the current game state.

```java
// 1. An NPC asset holds the pre-compiled expression object.
BuilderExpressionDynamicString dialogueExpression = npc.getAsset().getDialogueExpression();

// 2. A new execution context is created for this specific interaction.
//    It is populated with relevant variables like 'player.name'.
ExecutionContext context = NpcInteractionContextFactory.create(npc, player);

// 3. The expression is executed to get the final string.
String finalDialogue = dialogueExpression.getString(context);

// 4. The result is used in game logic.
world.sendChatMessage(npc, finalDialogue);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BuilderExpressionDynamicString()`. These objects are the output of the asset compilation pipeline. Attempting to create them manually without the correct instruction sequence will fail.
- **Context Reuse:** Do not reuse an `ExecutionContext` for different expression evaluations without resetting it. The context's internal stack is stateful and will be corrupted.
- **Cross-Thread Context Sharing:** Never pass the same `ExecutionContext` instance to calls of `getString` on different threads. This will cause severe concurrency issues.

## Data Pipeline
The creation and use of this class follows a distinct two-phase pipeline: a load-time compilation phase and a runtime evaluation phase.

> **Load-Time (Compilation)**:
> NPC Asset File (`"Hello, " + target.name`) -> ExpressionCompiler -> `Instruction[]` Bytecode -> **new BuilderExpressionDynamicString()**

> **Runtime (Evaluation)**:
> Game Event -> `ExecutionContext` (with `target.name` = "Player1") -> `getString(context)` on **BuilderExpressionDynamicString** instance -> Internal VM Execution -> Result ("Hello, Player1") -> Game Logic<ctrl63>

