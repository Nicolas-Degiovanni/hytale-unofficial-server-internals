---
description: Architectural reference for ValueType
---

# ValueType

**Package:** com.hypixel.hytale.server.npc.util.expression
**Type:** Utility

## Definition
```java
// Signature
public enum ValueType {
```

## Architecture & Concepts
The ValueType enum establishes the fundamental, static type system for the server-side NPC expression language. It acts as the canonical definition for all data types that can be manipulated within NPC scripts, behaviors, and logic evaluation. This component is not a data processor; rather, it is a schema that enables type safety and validation within a dynamic scripting environment.

The primary architectural role of ValueType is to provide metadata to the expression parser and interpreter. By classifying every value and variable, the system can perform compile-time or run-time type checking, preventing logical errors and crashes that would result from invalid operations, such as attempting to perform arithmetic on a STRING.

The inclusion of array types, and specifically the EMPTY_ARRAY type, indicates a design that supports collection-based logic while maintaining a degree of type flexibility. The system allows an empty array to be treated as a placeholder that is compatible with any typed array, deferring its concrete type until elements are added.

## Lifecycle & Ownership
- **Creation:** ValueType constants are instantiated by the Java Virtual Machine (JVM) during class loading. As an enum, its instances are created once and are managed entirely by the JVM.
- **Scope:** The instances of this enum are global singletons that persist for the entire lifetime of the server application. They are effectively static constants.
- **Destruction:** The enum and its constants are garbage collected only when the application's ClassLoader is unloaded, which typically occurs during a full server shutdown. Application code should never attempt to manage its lifecycle.

## Internal State & Concurrency
- **State:** ValueType is immutable. Each enum constant is a fixed, named value with no internal state that can be modified at runtime.
- **Thread Safety:** This class is inherently thread-safe. The enum constants are immutable, and the static utility methods are pure functions—their output depends solely on their input arguments, and they have no side effects. It is safe to access and use ValueType from any thread without synchronization.

## API Surface
The primary API consists of the enum constants themselves, which are used for type identity and comparison. The static methods provide core type-checking logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isAssignableType(from, to) | static boolean | O(1) | Determines if a value of type *from* can be safely assigned to a variable of type *to*. This is the core of the type-checking logic, with special handling for EMPTY_ARRAY polymorphism. |
| isTypedArray(valueType) | static boolean | O(1) | A utility check to determine if the given type is one of the concrete array types (NUMBER_ARRAY, STRING_ARRAY, BOOLEAN_ARRAY). |

## Integration Patterns

### Standard Usage
ValueType is used by the expression evaluation engine to validate operations. It should be used to check types before attempting to cast or operate on expression results.

```java
// Example: A type checker validating an assignment operation
ValueType variableType = ValueType.NUMBER_ARRAY;
ValueType valueType = expression.evaluate().getType(); // Returns EMPTY_ARRAY

// Correctly use the API to check for compatibility
if (ValueType.isAssignableType(valueType, variableType)) {
    // Assignment is valid
    assign(variable, value);
} else {
    throw new TypeMismatchException(...);
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Type Comparison:** Avoid writing complex switch statements or if-else chains to replicate the logic of isAssignableType. The built-in method correctly handles special cases like EMPTY_ARRAY.
- **Ignoring VOID:** Do not forget to handle the VOID type. An expression that resolves to VOID cannot be assigned to any variable, and this case must be checked explicitly. The isAssignableType method correctly returns false if either type is VOID.

## Data Pipeline
ValueType does not process a data stream itself. Instead, it provides the metadata used to validate the data types at a specific stage within the NPC script evaluation pipeline.

> Flow:
> NPC Script Text -> Parser -> Abstract Syntax Tree (AST) -> **Type Checker (using ValueType)** -> Interpreter -> Game World Mutation

