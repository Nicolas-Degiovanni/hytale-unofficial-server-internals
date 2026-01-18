---
description: Architectural reference for ExecutionContext
---

# ExecutionContext

**Package:** com.hypixel.hytale.server.npc.util.expression
**Type:** Transient

## Definition
```java
// Signature
public class ExecutionContext {
```

## Architecture & Concepts

The ExecutionContext is the core runtime engine for a stack-based virtual machine. It is designed to evaluate pre-compiled sequences of instructions, forming the heart of the server-side expression language used for dynamic game logic such as NPC behaviors, quest triggers, and combat calculations.

Architecturally, this class acts as an interpreter for a low-level bytecode-like format represented by the Instruction functional interface. It decouples the static logic of an expression (the instruction list) from the dynamic state of the game world (the Scope).

The system operates on a classic stack machine model:
1.  An expression (e.g., "player.health > 50 && npc.isHostile") is parsed and compiled into a linear sequence of Instruction objects.
2.  At runtime, an ExecutionContext is created.
3.  A Scope, which provides access to live game state variables and functions, is attached to the context.
4.  The execute method iterates through the instruction sequence. Each instruction manipulates the context's internal operand stack (pushing values, performing arithmetic, calling functions) to evaluate the expression.

This design provides high performance at runtime by avoiding the overhead of traversing an Abstract Syntax Tree (AST). The evaluation is a simple, linear pass over a pre-compiled instruction list.

### Lifecycle & Ownership

-   **Creation:** An ExecutionContext is a short-lived object, instantiated on-demand by a higher-level system responsible for evaluating a specific expression. It is typically created immediately before an evaluation is required.
-   **Scope:** The object's lifetime is bound to a single, complete execution of an instruction sequence. While the instance can be reused, the typical pattern is to create a new context for each distinct expression evaluation to ensure a clean state.
-   **Destruction:** The object holds no native resources and does not require explicit cleanup. It becomes eligible for garbage collection as soon as the caller discards the reference, which is usually after the execute method returns its final value.

## Internal State & Concurrency

-   **State:** The ExecutionContext is highly mutable and stateful. Its primary state is contained within the operandStack array and the stackTop integer, which tracks the stack pointer. It also holds references to external state via the Scope, combatConfig, and interactionVars fields. The operand stack grows dynamically as needed during execution.

-   **Thread Safety:** This class is **not thread-safe** and must be confined to a single thread. The internal state, particularly the operand stack and stack pointer, is manipulated extensively without any synchronization mechanisms. Concurrent calls to execute on the same instance will result in stack corruption, unpredictable behavior, and catastrophic failures.

    **WARNING:** Never share an ExecutionContext instance across threads. Each thread requiring expression evaluation must use its own dedicated instance. Similarly, the associated Scope object must not be modified by another thread during an active execution.

## API Surface

The public API is designed for two distinct clients: the expression evaluation system and the Instruction implementations themselves.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(instructions, scope) | ValueType | O(N) | Primary entry point. Executes a list of N instructions against a given scope. Resets internal state before execution. |
| push(value) | void | O(1) | Pushes a value of a specific type onto the operand stack. May trigger stack resizing. |
| popNumber() | double | O(1) | Pops the top operand from the stack, expecting it to be a number. |
| popBoolean() | boolean | O(1) | Pops the top operand from the stack, expecting it to be a boolean. |
| popString() | String | O(1) | Pops the top operand from the stack, expecting it to be a string. |
| getNumber(index) | double | O(1) | Peeks at a number value on the stack at a relative index from the top without popping it. |
| genPUSH(value) | Instruction | O(1) | Static factory method to create an instruction that pushes a constant value. |
| genREAD(ident, type, scope) | Instruction | O(1) | Static factory method to create an instruction that reads a variable from the scope. |
| genCALL(ident, numArgs, scope) | Instruction | O(1) | Static factory method to create an instruction that calls a function from the scope. |

## Integration Patterns

### Standard Usage

The intended use involves a compiler that produces an instruction list, and a runtime that creates a context, attaches a scope, and executes the list.

```java
// 1. A compiler or parser generates the instruction sequence once.
List<ExecutionContext.Instruction> compiledExpression = new ArrayList<>();
compiledExpression.add(ExecutionContext.genREAD("playerHealth", ValueType.NUMBER, null));
compiledExpression.add(ExecutionContext.genPUSH(50.0));
compiledExpression.add(ExecutionContext.GREATER); // playerHealth > 50

// 2. At runtime, create a context and a scope with live data.
ExecutionContext context = new ExecutionContext();
Scope currentScope = createPlayerScope(player); // Method that populates a Scope

// 3. Execute the compiled instructions.
ValueType resultType = context.execute(compiledExpression, currentScope);
if (resultType == ValueType.BOOLEAN && context.popBoolean()) {
    // Game logic branch
}
```

### Anti-Patterns (Do NOT do this)

-   **Reusing Across Threads:** Do not pass an ExecutionContext instance to another thread or use a shared instance from a thread pool. This will lead to state corruption. Create a new instance for each thread.
-   **Executing Without a Scope:** Calling an execute variant without first setting a Scope will result in a NullPointerException if any instruction attempts to read a variable or call a function.
-   **State Leakage:** Although an instance can be reused, be aware that fields like combatConfig and interactionVars persist between executions. Failure to reset or re-initialize these fields for a new evaluation can lead to subtle and hard-to-diagnose bugs. The recommended pattern is create-execute-discard.

## Data Pipeline

The ExecutionContext is a central processing stage in the server's dynamic logic pipeline.

> Flow:
> NPC Behavior File (String Expression) -> Expression Compiler -> **List of Instructions** -> ExecutionContext (with live game Scope) -> **Result Value** -> Game Logic System (e.g., AI State Machine)

