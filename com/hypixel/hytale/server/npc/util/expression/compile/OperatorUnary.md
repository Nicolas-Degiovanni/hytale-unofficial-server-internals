---
description: Architectural reference for OperatorUnary
---

# OperatorUnary

**Package:** com.hypixel.hytale.server.npc.util.expression.compile
**Type:** Utility / Registry

## Definition
```java
// Signature
public class OperatorUnary {
```

## Architecture & Concepts

The OperatorUnary class is a foundational component of the server-side NPC expression language compiler. It functions as a static, immutable registry that defines the complete set of valid unary operations (e.g., negation, logical not) available in the language.

Its primary role is to act as a bridge between the *syntactic analysis* (parsing tokens) and *semantic analysis* (type checking and code generation) phases of compilation. When the parser encounters a unary operator token, it consults this registry via the findOperator method to validate the operation.

This class effectively decouples the parser from the underlying virtual machine's instruction set. It provides a metadata layer that maps a high-level language construct (an operator symbol and an operand type) to a low-level, executable instruction. The registry contains all information needed for the compiler to:
1.  Verify that an operation is valid for a given data type (e.g., logical NOT is only valid for booleans).
2.  Determine the resulting data type of the operation for further type checking.
3.  Generate the correct bytecode-equivalent instruction for the ExecutionContext.

The design uses a static array initialized at class-loading time, ensuring that the set of operators is fixed and cannot be modified at runtime. This provides performance benefits and guarantees compiler behavior consistency.

### Lifecycle & Ownership
-   **Creation:** All OperatorUnary instances are created internally and exclusively during static class initialization. The private constructor and factory method `of` prevent any external instantiation. The `operators` array, which holds these instances, is populated once when the JVM loads the class.
-   **Scope:** The instances and the static registry exist for the entire lifetime of the server application. They are application-scoped singletons managed by the JVM's class loader.
-   **Destruction:** The objects are garbage collected only when the server shuts down and its class loader is unloaded. There is no manual cleanup required.

## Internal State & Concurrency
-   **State:** The OperatorUnary class is stateless from an external perspective. Internally, it manages a static array of immutable OperatorUnary objects. Once the `operators` array is initialized, its contents are never modified. This makes the entire structure deeply immutable.
-   **Thread Safety:** This class is unconditionally thread-safe. The internal registry is immutable and all public access is through a static, read-only search method (`findOperator`). No synchronization is required, and it can be safely used by multiple compiler threads concurrently without risk of data corruption or race conditions.

## API Surface

The primary interaction point is the static `findOperator` method. Instance methods are called on the object returned by this lookup.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| findOperator(token, type) | OperatorUnary | O(N) | **Primary Lookup Method.** Searches the internal registry for an operator matching the token and operand type. N is the small, fixed number of defined operators. Returns null if no valid operator is found. |
| getResultType() | ValueType | O(1) | Returns the ValueType produced by this operation. Crucial for the compiler's type-checking phase. |
| getCodeGen() | Function | O(1) | Returns the code generation function that produces an ExecutionContext.Instruction. This is the link to the virtual machine's instruction set. |
| hasCodeGen() | boolean | O(1) | Returns false if the operator is a no-op (e.g., unary plus) that requires no instruction to be emitted. |

## Integration Patterns

### Standard Usage

The intended consumer of this class is the expression compiler's semantic analysis or code generation stage. The typical flow involves parsing an expression, determining the operand's type, and then using OperatorUnary to validate the operation and retrieve the corresponding instruction generator.

```java
// Pseudo-code for a compiler's expression handler
Token operatorToken = parser.consumeToken(); // e.g., Token.LOGICAL_NOT
ValueType operandType = compileSubExpression(); // e.g., ValueType.BOOLEAN

// Look up the operator definition
OperatorUnary operator = OperatorUnary.findOperator(operatorToken, operandType);

if (operator == null) {
    // CRITICAL: This is a type error in the user's script.
    throw new CompilationException("Invalid operator " + operatorToken + " for type " + operandType);
}

// If valid, emit the instruction
if (operator.hasCodeGen()) {
    Function<Scope, ExecutionContext.Instruction> generator = operator.getCodeGen();
    ExecutionContext.Instruction instruction = generator.apply(currentScope);
    byteCodeBuilder.add(instruction);
}

// The result type is now known for the next stage of compilation
ValueType resultType = operator.getResultType();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is private. Attempting to create instances via reflection will break the integrity of the compiler and is strictly forbidden. The static registry is the single source of truth.
-   **Ignoring Null Return:** The `findOperator` method returns null to signify an invalid operation (a type error). Failure to check for a null return will result in a NullPointerException and crash the compiler thread. Always handle the null case as a compilation failure.
-   **Runtime Modification:** Do not attempt to modify the static `operators` array via reflection. The compiler's behavior relies on this registry being immutable and fixed at compile time.

## Data Pipeline

OperatorUnary sits at a critical junction in the compilation pipeline, translating parsed symbols into executable logic.

> Flow:
> NPC Script Text -> Lexer -> **Token** (e.g., UNARY_MINUS) -> Parser -> **ValueType** (of operand) -> **OperatorUnary.findOperator** -> Code Generator -> **ExecutionContext.Instruction** -> VM Execution

