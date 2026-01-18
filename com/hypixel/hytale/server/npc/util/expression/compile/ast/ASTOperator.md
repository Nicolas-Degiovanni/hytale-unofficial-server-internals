---
description: Architectural reference for ASTOperator
---

# ASTOperator

**Package:** com.hypixel.hytale.server.npc.util.expression.compile.ast
**Type:** Structural Component / Transient Node

## Definition
```java
// Signature
public abstract class ASTOperator extends AST {
```

## Architecture & Concepts
The ASTOperator is an abstract base class representing an operation within the server's expression language Abstract Syntax Tree (AST). It serves as a composite node, holding other AST nodes (its operands) as children. This class and its concrete subclasses, such as ASTOperatorUnary and ASTOperatorBinary, are fundamental to structuring the logical hierarchy of a parsed expression.

During the compilation phase, the Parser constructs a tree of AST nodes. ASTOperator nodes form the internal branches of this tree, while literal values (ASTValue) and variable references form the leaves. The primary responsibility of an ASTOperator is to orchestrate the generation of virtual machine instructions. It does this by first ensuring its operands generate their code, then contributing its own operation-specific instruction to the final bytecode list.

This class is a critical link between the syntactic analysis performed by the Parser and the semantic, code-generating phase of the expression compiler.

### Lifecycle & Ownership
-   **Creation:** ASTOperator instances are never created directly. They are instantiated by the expression compiler's Parser via the static factory method `fromParsedOperator`. This factory inspects a parsed token and delegates construction to the appropriate concrete subclass (unary or binary). The CompileContext, which manages the overall compilation task, is the effective owner of the AST.
-   **Scope:** The lifetime of an ASTOperator is strictly confined to a single expression compilation cycle. It is created during parsing, traversed during code generation, and then becomes eligible for garbage collection. It holds no state that persists beyond the compilation of its source expression.
-   **Destruction:** Destruction is implicit and managed by the Java Garbage Collector. Once the CompileContext completes its work and drops its reference to the root of the AST, the entire tree, including all its ASTOperator nodes, is reclaimed.

## Internal State & Concurrency
-   **State:** The core state of an ASTOperator is its internal `arguments` list, which holds its child operands. This list is mutable and is populated by the Parser during tree construction via the `addArgument` method. After the parsing phase is complete, the AST is treated as effectively immutable.
-   **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. The entire expression compilation pipeline, from parsing to code generation, must be executed on a single thread. The internal state, particularly the `arguments` list, is not protected against concurrent modification.

    **Warning:** Sharing an AST or its nodes across multiple threads will lead to race conditions and unpredictable compiler behavior.

## API Surface
The public API is designed for internal use by the expression compiler system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addArgument(AST argument) | void | O(1) | Adds a child operand to this operator's internal list. Establishes the parent-child link in the AST. |
| getArguments() | List<AST> | O(1) | Returns a direct reference to the list of child operands. |
| genCode(List, Scope) | ValueType | O(N) | Recursively generates bytecode for all child nodes, then for this operator. N is the number of nodes in the subtree. |
| fromParsedOperator(Parser.ParsedToken, CompileContext) | static void | O(1) | The primary factory method used by the Parser to create and integrate a new operator node into the AST. |

## Integration Patterns

### Standard Usage
This class is an internal component of the expression compiler and is not intended for direct use in game logic. Its usage is orchestrated entirely by the Parser and CompileContext during expression evaluation. The `fromParsedOperator` method is the sole entry point for its creation.

```java
// This is a conceptual example of the compiler's internal logic.
// Do not replicate this pattern.

// Inside the Parser loop...
Parser.ParsedToken operatorToken = consumeNextOperatorToken();
CompileContext context = getActiveCompileContext();

// The factory method handles creation and integration into the context's AST.
ASTOperator.fromParsedOperator(operatorToken, context);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new` on an ASTOperator subclass. The static factory `fromParsedOperator` contains critical logic for selecting the correct implementation (unary vs. binary) and integrating it into the compiler's state.
-   **Post-Parse Modification:** Do not call `addArgument` or otherwise modify the node's state after the parsing phase is complete. The AST is considered immutable once code generation begins. Modifying it can lead to invalid bytecode.
-   **Reusing Nodes:** Do not attempt to reuse ASTOperator nodes across different compilations. They are transient objects tied to a specific compilation context and scope.

## Data Pipeline
The ASTOperator acts as a structural and code-generating stage in the expression compilation pipeline.

> Flow:
> Raw Expression String -> Tokenizer -> Parser -> **ASTOperator.fromParsedOperator** -> AST Tree Construction -> `genCode()` Traversal -> Final `ExecutionContext.Instruction` List

