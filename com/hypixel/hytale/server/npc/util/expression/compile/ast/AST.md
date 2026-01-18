---
description: Architectural reference for the Abstract Syntax Tree (AST) base class.
---

# AST

**Package:** com.hypixel.hytale.server.npc.util.expression.compile.ast
**Type:** Structural Component

## Definition
```java
// Signature
public abstract class AST {
```

## Architecture & Concepts
The AST class is the abstract base for all nodes within an Abstract Syntax Tree, which is a core data structure in the NPC expression compilation pipeline. It serves as the intermediate representation between the syntactic analysis phase (parsing) and the code generation phase.

After the Tokenizer converts a raw expression string into a stream of Tokens, the Parser consumes these tokens to build a hierarchical tree of AST nodes. This tree represents the expression's logical structure and operator precedence, not just its textual sequence. For example, the expression `2 + 3 * 4` would be parsed into a tree with the `+` operation at the root, having `2` as one child and a subtree for `3 * 4` as the other.

Each concrete subclass of AST, such as NumberAST or BinaryOpAST, represents a specific language construct (a literal, an operator, a function call). The primary responsibility of an AST node is to translate its part of the logical tree into one or more executable instructions via the genCode method.

## Lifecycle & Ownership
- **Creation:** AST nodes are instantiated exclusively by the expression Parser during the compilation of a script. The Parser constructs the entire tree in memory as it consumes the token stream.
- **Scope:** Transient. An AST and its entire tree structure exist only for the duration of a single expression compilation. It is a short-lived, intermediate artifact.
- **Destruction:** The root AST node, and thereby the entire tree, becomes eligible for garbage collection as soon as the code generation phase is complete and the resulting list of ExecutionContext.Instruction objects has been returned. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Primarily immutable after construction. An AST node holds a reference to its originating Token and its resulting ValueType. The parent field is mutable and is set by the Parser during tree construction. The codeGen function is also mutable, assigned during a later compilation pass before final code generation.

- **Thread Safety:** **Not thread-safe.** The construction, modification, and traversal of an AST must be performed by a single thread. The entire expression compilation pipeline is designed for synchronous, single-threaded execution. Accessing or modifying an AST from multiple threads will result in a corrupt tree and unpredictable compiler behavior.

## API Surface
The public contract of the base AST class is designed for tree traversal and code generation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setParent(AST parent) | void | O(1) | Sets the parent of this node. Critical for tree construction. |
| getValueType() | ValueType | O(1) | Returns the data type this node will produce when executed. |
| getToken() | Token | O(1) | Returns the source token that generated this node. Used for error reporting. |
| isConstant() | boolean | O(N) | Abstract. Determines if the node and its children evaluate to a constant value. |
| asOperand() | ExecutionContext.Operand | O(1) | Throws by default. Overridden by nodes that represent constant values. |
| genCode(list, scope) | ValueType | O(1) | Generates and appends one or more instructions to the provided list. |

## Integration Patterns

### Standard Usage
Direct interaction with AST nodes is an internal function of the compiler. A higher-level system initiates compilation, which internally builds and processes the tree. Developers using the expression engine will not, and should not, interact with this class.

```java
// Conceptual example of internal compiler logic
List<Token> tokens = tokenizer.tokenize("5 > 3");
Parser parser = new Parser(tokens);
AST rootNode = parser.parseExpression(); // Parser creates the tree

List<ExecutionContext.Instruction> instructions = new ArrayList<>();
Scope scope = new Scope();
rootNode.genCode(instructions, scope); // Traverses the tree to generate code
```

### Anti-Patterns (Do NOT do this)
- **Manual Construction:** Do not attempt to build an AST manually. The tree's structure is guaranteed by the Parser; manual creation bypasses validation and will lead to invalid instruction sequences.
- **State Mutation After Parsing:** Modifying the AST (e.g., by calling setParent) after the Parser has finished its work is unsupported and will corrupt the compiler's state.
- **Caching or Reusing Trees:** An AST is a single-use object for one specific compilation. Do not cache it. If an expression needs to be re-compiled, it must be re-parsed to generate a new, clean tree.

## Data Pipeline
The AST is the central structure that transforms a syntactic representation into an executable one.

> Flow:
> Raw String -> Tokenizer -> List of Tokens -> Parser -> **AST Tree** -> Code Generator Traversal -> List of Instructions

