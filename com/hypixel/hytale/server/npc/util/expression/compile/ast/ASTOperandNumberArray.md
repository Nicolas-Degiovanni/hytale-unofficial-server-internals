---
description: Architectural reference for ASTOperandNumberArray
---

# ASTOperandNumberArray

**Package:** com.hypixel.hytale.server.npc.util.expression.compile.ast
**Type:** Transient

## Definition
```java
// Signature
public class ASTOperandNumberArray extends ASTOperand {
```

## Architecture & Concepts
ASTOperandNumberArray is a specialized node type within the Abstract Syntax Tree (AST) used by the server-side NPC expression compiler. Its sole purpose is to represent a *compile-time constant* array of double-precision floating-point numbers.

This class acts as a structural element during the parsing phase. When the compiler encounters a number array literal (e.g., `[1.0, 5.5, -3.0]`) or a reference to a predefined constant array variable, it instantiates this class to represent that value within the tree.

The key architectural principle is **immutability and compile-time resolution**. An ASTOperandNumberArray node does not represent a variable to be looked up at runtime; it *is* the value itself, embedded directly into the AST. During the final code generation step, this node produces a single, efficient instruction for the expression's virtual machine: push the constant array onto the operand stack.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the expression language parser during the compilation of an expression string. The various constructors accommodate different parsing scenarios, such as processing a literal array declaration or resolving a named constant from the current scope.
- **Scope:** The lifetime of an ASTOperandNumberArray instance is strictly limited to the duration of a single expression compilation. It is an ephemeral object that exists only as part of the in-memory AST.
- **Destruction:** The object is marked for garbage collection once the AST has been fully processed and translated into its final executable representation (e.g., bytecode). It holds no external resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** Immutable. The core state is the `constantNumberArray` field, a `double[]`, which is marked as `final`. It is populated once during construction and cannot be changed thereafter. All public methods are pure functions that read this state.
- **Thread Safety:** This class is inherently thread-safe due to its immutable design. An instance can be safely read from multiple threads without any synchronization mechanisms. In practice, it is almost always confined to the single thread performing the expression compilation.

## API Surface
The public API is minimal, reflecting its role as an internal data structure for the compiler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isConstant() | boolean | O(1) | Confirms that the node represents a compile-time constant. This implementation always returns true. |
| asOperand() | ExecutionContext.Operand | O(N) | Converts the compile-time AST node into a runtime operand wrapper. Involves allocating a new Operand and copying the array data. N is the number of elements in the array. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is an internal component of the expression compilation pipeline, managed entirely by the parser. A developer's interaction is indirect, through the writing of an expression string.

```java
// A developer does NOT interact with this class directly.
// The following is a conceptual illustration of the compiler's internal behavior.

// 1. A developer defines an expression.
String npcExpression = "processData([10.0, 20.0, 30.0])";

// 2. The compiler parses this string. When it encounters the array literal,
//    it internally constructs an ASTOperandNumberArray to represent it in the AST.
//
// PSEUDO-CODE of compiler internals:
// Token arrayToken = ...; // Token for '['
// double[] values = {10.0, 20.0, 30.0};
// AST node = new ASTOperandNumberArray(arrayToken, pos, values);
// ast.addNode(node);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of ASTOperandNumberArray using `new`. The constructors are tightly coupled to the compiler's internal state, such as the operand stack and token stream. Manual creation will lead to an invalid AST.
- **External State Modification:** Do not modify the contents of a `double[]` after passing it to the constructor. While the class's reference to the array is `final`, the array's contents are not. The entire expression engine relies on the assumption that this value is deeply constant.

## Data Pipeline
ASTOperandNumberArray is a key intermediate representation in the transformation of a raw expression string into executable code.

> Flow:
> Expression String -> Tokenizer -> Parser -> **ASTOperandNumberArray** (Node Creation) -> Code Generator -> Executable Bytecode

