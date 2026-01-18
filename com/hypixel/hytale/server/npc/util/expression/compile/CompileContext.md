---
description: Architectural reference for CompileContext
---

# CompileContext

**Package:** com.hypixel.hytale.server.npc.util.expression.compile
**Type:** Transient

## Definition
```java
// Signature
public class CompileContext implements Parser.ParsedTokenConsumer {
```

## Architecture & Concepts

The CompileContext is the central orchestrator for the expression compilation pipeline. It serves as a stateful, short-lived environment that transforms a raw string expression into a list of executable instructions, effectively a form of bytecode for the server's expression engine.

Its primary architectural role is to bridge the gap between the lexical and syntactical analysis performed by the Parser and the final code generation step. It implements the `Parser.ParsedTokenConsumer` interface, allowing it to act as a direct callback target. As the Parser consumes tokens from the input string, it notifies the CompileContext, which then builds an Abstract Syntax Tree (AST) in memory.

The core of this process relies on a stack-based algorithm. The `operandStack` field is used to construct the AST. Operands (values, variables) are pushed directly onto the stack. When an operator is processed, it consumes the required number of operands from the stack, combines them into a new AST node representing the operation, and pushes that new node back onto the stack. This process continues until the entire expression is parsed, leaving a single root AST node on the stack.

Finally, the CompileContext triggers the code generation phase by calling `genCode` on the root AST node. This traversal emits a linear sequence of `ExecutionContext.Instruction` objects, which can be executed by the `ExecutionContext` virtual machine.

## Lifecycle & Ownership

-   **Creation:** A new CompileContext is instantiated on-demand for each individual compilation task. It is not a shared service or a singleton. A higher-level system, such as an NPC behavior loader or a script manager, is responsible for its creation.
-   **Scope:** The lifetime of a CompileContext instance is strictly limited to a single invocation of one of its `compile` methods. All internal state, including the operand stack and instruction list, is transient and relevant only for the duration of that specific compilation.
-   **Destruction:** The object is eligible for garbage collection as soon as the `compile` method returns and the caller has retrieved the resulting instructions or value type. There are no external resources to release, and no explicit cleanup is required.

## Internal State & Concurrency

-   **State:** The CompileContext is highly mutable and inherently stateful. Its primary purpose is to accumulate state—the AST on the `operandStack` and the bytecode in the `instructions` list—throughout the parsing process. The `compile` methods explicitly clear this internal state at the beginning of each operation to ensure a clean slate.
-   **Thread Safety:** **This class is not thread-safe.** An instance must never be shared across threads or used for concurrent compilations. Its internal state, particularly the `operandStack`, is not protected by locks and would be immediately corrupted by simultaneous access, leading to unpredictable and incorrect compilation results. Each thread requiring compilation must create its own dedicated CompileContext instance.

## API Surface

The public API is designed for a single, focused purpose: compiling an expression. The `Parser.ParsedTokenConsumer` methods, while public, are considered an internal contract for the Parser and should not be called directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| compile(expression, scope, fullResolve) | ValueType | O(N) | Compiles the given string expression. N is the number of tokens. Throws IllegalStateException on syntax or semantic errors. |
| getInstructions() | List | O(1) | Returns the list of generated instructions after a successful compilation. |
| getResultType() | ValueType | O(1) | Returns the final data type of the compiled expression. |
| checkResultType(type) | void | O(1) | Validates that the compiled result type matches the expected type. Throws IllegalStateException on mismatch. |

## Integration Patterns

### Standard Usage

The intended use is to create a new instance, invoke the compile method with the expression and scope, and then retrieve the generated instructions for later execution.

```java
// A new context is created for this specific compilation task.
CompileContext compiler = new CompileContext();
Scope npcScope = new Scope(); // Assume this is populated with NPC variables

// The list to hold the output bytecode.
List<ExecutionContext.Instruction> instructions = new ObjectArrayList<>();

// The compile method populates the provided instruction list.
ValueType result = compiler.compile("npc.health > 50", npcScope, true, instructions);

// The 'instructions' list now contains the executable program.
// It can be passed to an ExecutionContext to be run.
```

### Anti-Patterns (Do NOT do this)

-   **Instance Reuse:** While the `compile` method resets internal state, it is poor practice to reuse a CompileContext instance across unrelated logical tasks. The design promotes a "create, use, discard" pattern.
-   **Concurrent Compilation:** Never pass a single CompileContext instance to multiple threads to perform compilations in parallel. This will corrupt the internal stack and produce invalid bytecode.
-   **Direct State Manipulation:** Do not manually call `pushOperand` or `processOperator`. These methods are part of the contract with the Parser and rely on the Parser's specific invocation order. Manipulating the internal stack directly will break the compilation algorithm.

## Data Pipeline

The CompileContext is a critical stage in the expression-to-bytecode pipeline. It receives a stream of parsed tokens and is responsible for semantic analysis and code generation.

> Flow:
> String Expression -> Parser (Tokenization) -> **CompileContext** (AST Construction via Callbacks) -> AST Traversal (Code Generation) -> `List<Instruction>` (Bytecode)

