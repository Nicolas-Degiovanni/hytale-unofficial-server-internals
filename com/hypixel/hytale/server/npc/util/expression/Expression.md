---
description: Architectural reference for Expression
---

# Expression

**Package:** com.hypixel.hytale.server.npc.util.expression
**Type:** Transient

## Definition
```java
// Signature
public class Expression {
```

## Architecture & Concepts
The Expression class is the primary facade for the server-side dynamic expression engine, commonly used for NPC behavior, scripting, and other game logic. It provides a unified interface for compiling human-readable string expressions into an intermediate bytecode format and subsequently executing that bytecode.

This architecture deliberately separates the two distinct phases of an expression's life:

1.  **Compilation:** An input string is processed by a static Lexer to produce a stream of Tokens. The instance-specific CompileContext then parses this stream, performing semantic analysis against a provided Scope to resolve variables and functions. The final output is a list of low-level Instruction objects, which is a platform-independent bytecode representation of the original logic.

2.  **Execution:** The generated Instruction list is fed into an ExecutionContext, which acts as a virtual machine. It interprets the bytecode, manipulating a stack and using a runtime Scope to fetch live data (e.g., a specific NPC's health).

This two-phase design is a critical performance optimization. It allows expensive compilation to be performed once (e.g., when a zone loads), and the resulting lightweight bytecode can be executed repeatedly and efficiently (e.g., every game tick for multiple NPCs).

## Lifecycle & Ownership
- **Creation:** An Expression object is instantiated directly via its constructor: `new Expression()`. It is a lightweight object intended for on-demand creation by systems that need to process expressions, such as a behavior tree runner or a quest manager.

- **Scope:** The lifetime of an Expression instance is typically short and task-oriented. Each instance encapsulates its own CompileContext and ExecutionContext. This state is reused across multiple calls on the *same instance* but is completely isolated from other Expression instances.

- **Destruction:** The object is managed by the Java garbage collector and is eligible for cleanup once all references to it are dropped. It does not hold any native resources and requires no explicit destruction method.

## Internal State & Concurrency
- **State:** The Expression class is stateful. Each instance maintains its own CompileContext and ExecutionContext. Methods like `compile` and `execute` mutate the internal state of these context objects. For example, after a call to `compile`, the internal instruction list of the CompileContext is populated.

- **Thread Safety:** **This class is fundamentally not thread-safe.** Sharing a single Expression instance across multiple threads is an unsupported and dangerous pattern. Concurrent calls to `compile` or `execute` on the same instance will result in race conditions, internal state corruption, and unpredictable behavior. Each thread requiring expression evaluation must create and use its own private Expression instance.

    **Warning:** The static methods, such as compileStatic, are thread-safe as they create temporary, method-scoped context objects. The shared Lexer returned by getLexerInstance is immutable and also safe for concurrent use.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| compile(expression, scope, instructions, fullResolve) | ValueType | O(N) | Compiles a string into a list of instructions. N is the length of the expression. This is the primary method for the compilation phase. |
| execute(instructions, scope) | ExecutionContext | O(M) | Executes a pre-compiled list of instructions. M is the number of instructions. Returns the context, which contains the final result. |
| evaluate(expression, scope) | ExecutionContext | O(N + M) | A convenience method that performs both compilation and execution in a single call. Should not be used in performance-critical loops. |
| compileStatic(expression, scope, instructions) | ValueType | O(N) | A stateless, thread-safe utility method to compile an expression without needing an Expression instance. |
| getLexerInstance() | Lexer | O(1) | Returns the globally shared, immutable Lexer instance for custom tokenization needs. |

## Integration Patterns

### Standard Usage
The most performant and intended usage pattern is to separate compilation from execution. Compile expressions once during an initialization phase and execute the resulting bytecode frequently during the game loop.

```java
// Phase 1: Initialization (e.g., on server start or world load)
Expression expressionEngine = new Expression();
Scope compileTimeScope = buildNpcScope(); // Scope with function definitions
List<ExecutionContext.Instruction> isAngryInstructions = new ObjectArrayList<>();
expressionEngine.compile("self.getHealth() < 50 && self.hasTarget()", compileTimeScope, isAngryInstructions);

// Phase 2: Game Loop (e.g., per-NPC, per-tick)
// The same compiled instructions can be run for many different NPCs
Scope runtimeScope = createRuntimeScopeForNpc(npc); // Scope with live NPC data
ExecutionContext resultContext = expressionEngine.execute(isAngryInstructions, runtimeScope);

if (resultContext.getResult().asBoolean()) {
    npc.enterFrenzyState();
}
```

### Anti-Patterns (Do NOT do this)
- **Using evaluate in Loops:** Never call `evaluate` inside a tight, performance-critical loop like a per-tick update. This forces the engine to re-tokenize and re-compile the same string on every iteration, causing significant performance degradation. Pre-compile the expression using `compile`.

- **Sharing Instances Across Threads:** Do not create a single `Expression` instance and share it between different threads or systems. This will lead to state corruption. Each thread must have its own `Expression` object.

- **Modifying Instruction Lists:** The `List<Instruction>` populated by `compile` should be treated as read-only. Modifying this list after compilation will lead to undefined behavior during execution.

## Data Pipeline
The flow of data through the Expression engine follows a clear, multi-stage pipeline from a raw string to a final computed value.

> Flow:
> Raw String -> **Expression.compile()** [Lexer -> Parser] -> Instruction List (Bytecode) -> **Expression.execute()** [VM] -> ExecutionContext (containing result)

