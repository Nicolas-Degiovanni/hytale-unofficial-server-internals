---
description: Architectural reference for BuilderExpressionStaticEmptyArray
---

# BuilderExpressionStaticEmptyArray

**Package:** com.hypixel.hytale.server.npc.asset.builder.expression
**Type:** Singleton

## Definition
```java
// Signature
public class BuilderExpressionStaticEmptyArray extends BuilderExpression {
```

## Architecture & Concepts
The BuilderExpressionStaticEmptyArray is a specialized node within the server-side NPC expression Abstract Syntax Tree (AST). It represents a literal, constant, and empty array value.

Architecturally, this class implements the **Flyweight** design pattern. It exists as a stateless singleton, identified by the public static final field INSTANCE. This design is critical for performance, as it prevents the repeated allocation and garbage collection of new empty array objects every time an empty array literal is encountered during the parsing and evaluation of NPC behavior scripts.

This component is part of the "builder" system, indicating its primary role is during the construction phase of an expression tree from a source asset, such as a JSON or script file. It provides a canonical, immutable representation for the concept of "an empty array of any type".

## Lifecycle & Ownership
- **Creation:** The single instance is created by the JVM class loader when the BuilderExpressionStaticEmptyArray class is first referenced. It is not instantiated by any engine system or factory.
- **Scope:** Application-scoped. The INSTANCE persists for the entire lifetime of the server process.
- **Destruction:** The object is eligible for garbage collection only when the server shuts down and its class loader is unloaded.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its methods always return references to pre-existing, static constants defined in ArrayUtil.
- **Thread Safety:** This class is inherently **thread-safe**. As a stateless singleton, it can be safely shared and accessed concurrently by multiple threads without any need for locks or synchronization. The expression evaluation engine can use the single INSTANCE across all NPC evaluation contexts simultaneously.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | ValueType | O(1) | Returns the constant ValueType.EMPTY_ARRAY. |
| isStatic() | boolean | O(1) | Returns true, indicating its value is constant and known before execution. |
| getNumberArray(ctx) | double[] | O(1) | Returns a shared, static empty double array. The ExecutionContext is ignored. |
| getIntegerArray(ctx) | int[] | O(1) | Returns a shared, static empty integer array. The ExecutionContext is ignored. |
| getStringArray(ctx) | String[] | O(1) | Returns a shared, static empty String array. The ExecutionContext is ignored. |
| getBooleanArray(ctx) | boolean[] | O(1) | Returns a shared, static empty boolean array. The ExecutionContext is ignored. |
| addToScope(name, scope) | void | O(1) | Registers a new constant variable in the provided scope with an empty array value. |
| updateScope(scope, name, ctx) | void | O(1) | Changes an existing variable in the provided scope to an empty array value. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay logic developers. It is an internal component of the expression parsing and compilation system. The system uses the singleton instance to represent an empty array literal within a compiled expression.

```java
// Inside an expression parser or builder
// When an empty array literal "[]" is found:
BuilderExpression emptyArrayNode = BuilderExpressionStaticEmptyArray.INSTANCE;

// The node is then added to the AST.
// Later, during evaluation:
int[] result = emptyArrayNode.getIntegerArray(someExecutionContext);
// result is now ArrayUtil.EMPTY_INT_ARRAY
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new BuilderExpressionStaticEmptyArray()`. This defeats the performance benefit of the singleton pattern and creates unnecessary objects. Always use the static `INSTANCE` field.
- **Attempting to Modify Returned Arrays:** The arrays returned by the `get...Array` methods are static constants. Attempting to modify their contents will lead to undefined behavior and affect all parts of the server that use these constants. They must be treated as read-only.

## Data Pipeline
This class acts as a static data source within the expression evaluation pipeline. It does not process incoming data; it originates a value.

> Flow:
> NPC Asset Parser (finds `[]` literal) -> AST Builder (uses **BuilderExpressionStaticEmptyArray.INSTANCE**) -> Expression Evaluator (calls `get...Array`) -> Static Empty Array Value

