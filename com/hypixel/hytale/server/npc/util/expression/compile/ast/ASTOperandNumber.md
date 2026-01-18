---
description: Architectural reference for ASTOperandNumber
---

# ASTOperandNumber

**Package:** com.hypixel.hytale.server.npc.util.expression.compile.ast
**Type:** Transient / Value Object

## Definition
```java
// Signature
public class ASTOperandNumber extends ASTOperand {
```

## Architecture & Concepts
The ASTOperandNumber class is a leaf node within the Abstract Syntax Tree (AST) representation of a parsed server-side expression. Its sole responsibility is to represent a compile-time, constant numeric value. This can be either a numeric literal (e.g., `10.5`) or a named constant (e.g., `MAX_HEALTH`) that is resolved during the compilation phase.

This class is a critical component of the expression compiler, acting as a terminal point in the tree structure for any arithmetic or logical operation. It embodies the principle of separating compile-time resolution from runtime evaluation.

Architecturally, it supports two distinct evaluation strategies:
1.  **Bytecode Generation:** The `codeGen` field, initialized at construction, holds a lambda function. This function generates a `PUSH` instruction for the expression engine's virtual machine (ExecutionContext), pushing the constant value onto the stack at runtime.
2.  **Direct Interpretation:** The `asOperand` method facilitates immediate, tree-walking evaluation by converting the node's value directly into an ExecutionContext.Operand, bypassing the bytecode step.

## Lifecycle & Ownership
-   **Creation:** An ASTOperandNumber is instantiated exclusively by the expression parser during the semantic analysis phase. It is created when the parser consumes a token representing either a numeric literal or an identifier that the `Scope` resolves as a constant number.
-   **Scope:** The object's lifetime is strictly limited to the compilation of a single expression. It exists only as a node within the in-memory AST.
-   **Destruction:** It is eligible for garbage collection as soon as the AST has been processed into its final form (e.g., bytecode) and is no longer referenced by the compiler.

## Internal State & Concurrency
-   **State:** Immutable. The internal `constantNumber` field is final and is set only once during construction. The object's state cannot be changed after creation.
-   **Thread Safety:** This class is inherently thread-safe due to its immutability. An instance can be safely read from multiple threads without any synchronization mechanisms. In practice, its usage is typically confined to the single thread performing the expression compilation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ASTOperandNumber(token, pos, value) | Constructor | O(1) | Creates a node from a literal double value. |
| ASTOperandNumber(token, pos, scope, id) | Constructor | O(log N) | Creates a node by resolving a constant identifier from the provided Scope. Throws IllegalArgumentException if the identifier is not a constant. |
| getNumber() | double | O(1) | Returns the raw double value of the constant. |
| isConstant() | boolean | O(1) | Always returns true, fulfilling its contract as a constant AST node. |
| asOperand() | ExecutionContext.Operand | O(1) | Creates a runtime operand wrapper for the constant value, used for direct interpretation. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is an internal component of the expression compilation pipeline, created and managed by the parser.

```java
// Example of internal creation by a parser
// Assume 'token' is a parsed numeric literal "42.0"
double value = Double.parseDouble(token.getText());
ASTNode numericNode = new ASTOperandNumber(token, token.getPosition(), value);

// The node is then attached to the parent AST branch
parentOperation.addChild(numericNode);
```

### Anti-Patterns (Do NOT do this)
-   **Representing Variables:** Do not attempt to use this class for runtime variables. It is designed exclusively for values known at compile time. Using the identifier-based constructor with a non-constant name will result in an exception.
-   **Post-Construction Modification:** Do not use reflection or other means to alter the `constantNumber` field after instantiation. The entire expression evaluation system relies on the immutability of this node.

## Data Pipeline
The ASTOperandNumber serves as a data container in the middle of the compilation pipeline, transforming a textual token into a structured, typed representation that can be converted into executable instructions.

> Flow:
> Expression String -> Tokenizer -> Parser -> **ASTOperandNumber (in AST)** -> Code Generator -> Bytecode -> ExecutionContext VM

