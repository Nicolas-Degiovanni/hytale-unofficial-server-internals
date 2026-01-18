---
description: Architectural reference for ASTOperandStringArray
---

# ASTOperandStringArray

**Package:** com.hypixel.hytale.server.npc.util.expression.compile.ast
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ASTOperandStringArray extends ASTOperand {
```

## Architecture & Concepts
The ASTOperandStringArray is a specialized node type within the Abstract Syntax Tree (AST) used by the server's expression language compiler. It serves a single, critical purpose: to represent a compile-time constant array of strings.

This class is a terminal node, or leaf, in the AST. Unlike operational nodes that represent actions like addition or function calls, this node represents a static value. During the parsing phase, the compiler creates instances of ASTOperandStringArray when it encounters either a string array literal (e.g., `["a", "b", "c"]`) or a named constant that resolves to a string array.

Its primary architectural function is to bridge the gap between the parsed source text and the executable bytecode. It holds the constant array value throughout the compilation process and provides the code generator with a pre-defined lambda, `codeGen`, which emits the necessary `PUSH` instruction to place the array onto the execution stack at runtime.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the expression language parser during the syntax analysis stage. The specific constructor invoked depends on the source token: a literal array definition will use the stack-based constructor, while a named constant will use the scope-based constructor.
- **Scope:** The lifetime of an ASTOperandStringArray instance is strictly limited to the compilation of a single expression. It exists only as a node within the in-memory AST for that expression.
- **Destruction:** The object is marked for garbage collection as soon as the AST is processed by the code generator and is no longer referenced. The final compiled bytecode contains the raw string array data, not a reference to this AST node.

## Internal State & Concurrency
- **State:** Immutable. The internal `constantStringArray` field is private, final, and initialized only once within the constructor. The `isConstant` method hard-codes a return value of `true`, enforcing this design contract. All operations on this object are read-only.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. No synchronization is required. While the expression compiler may operate in a single-threaded context per-expression, this object could be safely read from multiple threads without risk of data corruption.

## API Surface
The public API is primarily composed of constructors used by the compiler internals. Direct interaction by developers is not an intended use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ASTOperandStringArray(token, pos, array) | constructor | O(1) | Core constructor. Directly wraps a pre-existing string array. |
| ASTOperandStringArray(token, pos, scope, id) | constructor | O(1) | Resolves a named constant from a Scope. Throws IllegalArgumentException if the identifier is not a constant. |
| ASTOperandStringArray(token, pos, stack, first, count) | constructor | O(N) | Builds the array from N other AST nodes on the operand stack. Used for parsing literals. |
| isConstant() | boolean | O(1) | Always returns true, indicating the value is a compile-time constant. |
| asOperand() | ExecutionContext.Operand | O(1) | Creates a runtime operand wrapper for the internal array. |

## Integration Patterns

### Standard Usage
This class is an internal component of the compiler and is not meant to be used directly. The standard integration is entirely managed by the expression processing pipeline. The parser generates these nodes, and the code generator consumes them.

```java
// PSEUDO-CODE: Compiler Internal Logic
// The parser encounters ["hello", "world"] and creates the node.
Stack<AST> operandStack = ...; // Contains ASTOperandString for "hello" and "world"
AST node = new ASTOperandStringArray(token, pos, operandStack, firstArg, 2);

// The code generator later visits the node.
// The node's internal 'codeGen' lambda is executed.
node.codeGen.generate(executionContext); // Emits PUSH ["hello", "world"]
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Manually creating an instance of ASTOperandStringArray outside of the compiler's parsing logic is an architectural violation. The node will not be part of a valid AST and will never be processed by the code generator.
- **Assuming Mutability:** Do not attempt to retrieve the array from this object and modify its contents. The entire expression system relies on the immutability of constant operands. Modifying it could lead to unpredictable behavior in shared constants.

## Data Pipeline
The ASTOperandStringArray is a key component in the transformation of source text into executable instructions. It acts as a temporary, structured container for a constant value during this translation.

> Flow:
> Expression String -> Tokenizer -> Parser -> **ASTOperandStringArray (in AST)** -> Code Generator -> Bytecode with PUSH instruction and constant data

