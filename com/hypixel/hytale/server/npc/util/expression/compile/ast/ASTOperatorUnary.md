---
description: Architectural reference for ASTOperatorUnary
---

# ASTOperatorUnary

**Package:** com.hypixel.hytale.server.npc.util.expression.compile.ast
**Type:** Transient

## Definition
```java
// Signature
public class ASTOperatorUnary extends ASTOperator {
```

## Architecture & Concepts
ASTOperatorUnary is a node type within the Abstract Syntax Tree (AST) used by the server-side expression language compiler. It specifically represents a unary operation, such as negation (`-x`) or logical not (`!y`). This class is a structural component, not a service.

Its primary role is to encapsulate a single unary operator and its corresponding operand (another AST node) during the parsing phase. The expression compiler, likely implementing a variant of the Shunting-yard algorithm, uses a stack-based approach to build the AST. When the parser encounters a unary operator token, it invokes the static factory method on this class to create the appropriate node, linking it to the operand that was just parsed.

A critical architectural feature is its support for **compile-time constant folding**. If the operand to a unary operator is a known constant, this class will evaluate the expression during the compilation phase itself, replacing the operator and operand nodes with a single, constant result node (ASTOperand). This is a significant performance optimization that reduces the number of instructions to be executed at runtime.

## Lifecycle & Ownership
- **Creation:** ASTOperatorUnary instances are created exclusively by the static factory method `fromUnaryOperator`. This method is invoked by the expression `Parser` when it consumes a token corresponding to a unary operator. It is never instantiated directly.
- **Scope:** The lifetime of an ASTOperatorUnary instance is extremely short and is strictly confined to the compilation of a single expression. It exists on the `operandStack` within the `CompileContext` only for the duration of the parsing process.
- **Destruction:** Once the AST is fully constructed and the corresponding runtime instructions are generated, the entire tree, including any ASTOperatorUnary nodes, is no longer referenced and becomes eligible for garbage collection. It does not persist beyond the compilation routine.

## Internal State & Concurrency
- **State:** The state of an ASTOperatorUnary instance is effectively immutable after construction. It holds final references to the operator's definition (`OperatorUnary`), the source token, and its single child AST node (the argument). It contains no mutable state or caches.
- **Thread Safety:** This class is **not thread-safe** and is not designed to be. The entire expression compilation process, including the manipulation of the `CompileContext` and its associated stacks, is designed to be executed by a single thread. Concurrent access to the `fromUnaryOperator` method with a shared `CompileContext` will lead to stack corruption and undefined behavior.

## API Surface
The public contract is minimal and intended for internal use by the compiler system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromUnaryOperator(operand, compileContext) | static void | O(1) | Factory method. Pops an operand from the stack, performs type checking and constant folding, and pushes the resulting AST node back onto the stack. Throws ParseException on type mismatch or insufficient operands. |
| isConstant() | boolean | O(1) | Returns false. Unary operations are considered non-constant by default, forcing runtime evaluation unless they are folded during compilation. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is an internal component of the expression parsing pipeline. The system interacts with it automatically.

```java
// Conceptual example of internal Parser logic
// This code does NOT exist in this class but illustrates its usage.

// When the Parser identifies a unary operator token...
Parser.ParsedToken unaryOpToken = ...;
CompileContext context = ...;

// It invokes the factory method to process the operator.
// The method internally manipulates the context's operand stack.
ASTOperatorUnary.fromUnaryOperator(unaryOpToken, context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ASTOperatorUnary(...)`. The static factory `fromUnaryOperator` contains essential logic for type checking, operator resolution, and constant folding. Bypassing it will result in an incomplete and incorrect AST.
- **State Mutation:** Do not attempt to modify an ASTOperatorUnary node after its creation. The AST is treated as an immutable structure once built.
- **Concurrent Compilation:** Do not share a `CompileContext` across multiple threads. The compilation process is inherently single-threaded.

## Data Pipeline
ASTOperatorUnary acts as a processor node during the transformation of a token stream into a final AST.

> Flow:
> Raw String -> `Parser` Token Stream -> **ASTOperatorUnary.fromUnaryOperator** -> Modified Operand Stack -> Final AST Root

