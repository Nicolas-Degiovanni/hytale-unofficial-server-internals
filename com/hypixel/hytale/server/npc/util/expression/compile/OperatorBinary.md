---
description: Architectural reference for OperatorBinary
---

# OperatorBinary

**Package:** com.hypixel.hytale.server.npc.util.expression.compile
**Type:** Utility

## Definition
```java
// Signature
public class OperatorBinary {
```

## Architecture & Concepts

The OperatorBinary class is a core component of the server-side NPC expression compilation engine. It functions as a static, immutable registry that defines the complete set of valid binary operations (e.g., addition, comparison, logical AND) available within the expression language.

Its primary architectural role is to decouple the expression *parser* from the *code generator*. The parser identifies tokens and expression structure, while the code generator produces executable instructions for the ExecutionContext virtual machine. OperatorBinary acts as the bridge between them, providing the semantic rules and corresponding instruction mapping for any given operator and its operand types.

Each OperatorBinary instance represents a single, unique operation, such as "adding a NUMBER to a NUMBER" or "comparing a BOOLEAN to a BOOLEAN". This design enforces strong type checking at compile time and provides a single, authoritative source for operator behavior. The entire system is pre-populated at class-loading time, ensuring that the expression language's capabilities are fixed and predictable.

### Lifecycle & Ownership

-   **Creation:** All OperatorBinary instances are created and stored in the static `operators` array during JVM class loading. The private constructor and static `of` factory method are used exclusively within this static initializer block. No new instances are ever created during the application's runtime.
-   **Scope:** As a static registry, the collection of OperatorBinary objects is application-scoped. It persists for the entire lifetime of the server process.
-   **Destruction:** The objects are garbage collected only when the `OperatorBinary` class is unloaded by the JVM, which typically occurs at application shutdown.

## Internal State & Concurrency

-   **State:** The OperatorBinary class is deeply immutable. Each instance's state (token, operand types, result type, and code generation function) is set once via its private constructor and never changes. The static `operators` array that holds these instances is also effectively immutable after its initial population.
-   **Thread Safety:** This class is unconditionally thread-safe. All data is immutable and all operations are read-only. The primary public method, `findOperator`, can be safely called by multiple compiler threads concurrently without any need for external synchronization or locks.

## API Surface

The public API is minimal, designed for lookup and data retrieval by the expression compiler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| findOperator(token, lhs, rhs) | OperatorBinary | O(N) | Looks up and returns the operator definition for the given token and operand types. Returns null if no match is found. Complexity is constant time as N is a small, fixed number. |
| getResultType() | ValueType | O(1) | Returns the type of the value that results from executing this operation. Used for compile-time type checking. |
| getCodeGen() | Function | O(1) | Returns the code generation function that produces the corresponding ExecutionContext.Instruction. |

## Integration Patterns

### Standard Usage

The `Compiler` component is the sole consumer of this class. During the code generation phase, after parsing a binary expression from an Abstract Syntax Tree (AST), the compiler uses `findOperator` to resolve the operation and emit the correct instruction.

**WARNING:** A null return from `findOperator` indicates a type error in the source expression (e.g., `"hello" + true`). The compiler must handle this by terminating compilation and reporting a type mismatch error.

```java
// Pseudo-code from a hypothetical Compiler class
void compileBinaryNode(BinaryNode node) {
    // Recursively compile left and right hand sides
    compile(node.getLhs()); // Pushes a value onto the stack
    compile(node.getRhs()); // Pushes another value onto the stack

    // Determine operand types from the compilation context
    ValueType lhsType = context.peekType(-2);
    ValueType rhsType = context.peekType(-1);
    Token token = node.getToken();

    // Look up the operation
    OperatorBinary op = OperatorBinary.findOperator(token, lhsType, rhsType);

    if (op == null) {
        throw new CompilationException("Type error: Cannot apply operator " + token
            + " to types " + lhsType + " and " + rhsType);
    }

    // Generate the instruction for the VM
    ExecutionContext.Instruction instruction = op.getCodeGen().apply(scope);
    emit(instruction);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** The constructor is private. Do not attempt to create instances of OperatorBinary via reflection. The system relies exclusively on the pre-defined static set of operators.
-   **State Modification:** Do not attempt to modify the static `operators` array via reflection. Doing so would corrupt the expression language definition for the entire server, leading to unpredictable behavior and crashes.

## Data Pipeline

OperatorBinary is not a data processor itself, but a critical lookup table used *within* the expression compilation pipeline. It validates and provides logic for transforming a parsed expression into an executable instruction.

> Flow:
> Expression String -> Parser (AST) -> **Compiler uses OperatorBinary** -> Code Generator -> Executable Instruction List

