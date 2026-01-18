---
description: Architectural reference for BuilderExpressionStaticBooleanArray
---

# BuilderExpressionStaticBooleanArray

**Package:** com.hypixel.hytale.server.npc.asset.builder.expression
**Type:** Transient Value Object

## Definition
```java
// Signature
public class BuilderExpressionStaticBooleanArray extends BuilderExpression {
```

## Architecture & Concepts
The BuilderExpressionStaticBooleanArray is a fundamental component of the server-side NPC expression engine. It serves as a leaf node within an expression tree, representing a compile-time constant value—specifically, an array of booleans.

Its primary architectural role is to encapsulate a literal `boolean[]` value, allowing it to be seamlessly integrated into the expression evaluation system. By implementing the BuilderExpression interface, it conforms to the contract required by the engine, but its `isStatic` method returns true. This signals to the expression evaluator that its value will never change during runtime, enabling significant performance optimizations such as pre-calculation and caching.

This class is a critical part of the data pipeline that transforms static asset definitions (JSON) into executable in-game logic for NPC behaviors.

### Lifecycle & Ownership
- **Creation:** Instances are created in two primary scenarios:
    1.  During asset deserialization, the static factory method `fromJSON` is invoked by the asset loader to parse a JSON array into a BuilderExpressionStaticBooleanArray object.
    2.  Programmatically by other expression builders or parsers that need to represent a literal boolean array within a dynamically constructed expression tree.
    A shared static instance, `INSTANCE_EMPTY`, is used to represent empty arrays, conserving memory.

- **Scope:** The object's lifetime is tied to the expression tree that contains it. It is a transient object, created for the purpose of evaluation and discarded once the parent expression is no longer needed.

- **Destruction:** Managed entirely by the Java garbage collector. There are no native resources or explicit cleanup procedures required.

## Internal State & Concurrency
- **State:** The core state is a single `final` field, `booleanArray`. This state is **effectively immutable**. While a Java array's contents are technically mutable, the class provides no methods to alter the array after construction. Consumers are expected to treat the returned array as read-only.

- **Thread Safety:** This class is **thread-safe**. Its immutable state guarantees that it can be safely read and evaluated by multiple threads concurrently without locks or synchronization. This is essential for the server's multi-threaded environment, where numerous NPC behaviors may be processed in parallel.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | ValueType | O(1) | Returns the constant `ValueType.BOOLEAN_ARRAY`. |
| isStatic() | boolean | O(1) | Returns `true`, indicating the value is a compile-time constant. |
| getBooleanArray(ctx) | boolean[] | O(1) | Returns the internal boolean array. The context is ignored. |
| addToScope(name, scope) | void | O(1) | Adds the internal array to the given scope under the specified name. |
| updateScope(scope, name, ctx) | void | O(1) | Updates a value in the given scope with the internal array. |
| fromJSON(jsonArray) | BuilderExpressionStaticBooleanArray | O(N) | Static factory to create an instance from a JSON array. Returns null on parse failure. |

## Integration Patterns

### Standard Usage
This class is not typically instantiated directly by gameplay logic developers. It is used by the underlying expression parsing and building system. The primary interaction is retrieving its value during evaluation.

```java
// Example of an evaluator processing an expression node
// Assume 'expression' is an instance of BuilderExpressionStaticBooleanArray

if (expression.isStatic()) {
    // The value can be safely cached
    boolean[] staticValue = expression.getBooleanArray(null);
    // ... use staticValue for further processing
}
```

### Anti-Patterns (Do NOT do this)
- **Modifying the Returned Array:** The most severe anti-pattern is modifying the array returned by `getBooleanArray`. This violates the class's immutability contract and will cause unpredictable, globally-impacting bugs, as the same instance may be shared across multiple expression trees.

    ```java
    // DO NOT DO THIS
    BuilderExpressionStaticBooleanArray expr = ...;
    boolean[] values = expr.getBooleanArray(context);
    values[0] = !values[0]; // This corrupts the static state for all users
    ```

- **Ignoring Null from fromJSON:** The `fromJSON` factory returns null if the input JSON is malformed (e.g., contains non-boolean values). Callers must perform a null check to prevent NullPointerExceptions in the asset loading pipeline.

## Data Pipeline
The class acts as a bridge between the raw data format (JSON) and the runtime expression engine.

> Flow:
> NPC Asset JSON File -> Gson Deserializer -> **BuilderExpressionStaticBooleanArray.fromJSON** -> Expression Tree Node -> ExpressionContext -> NPC Behavior Logic

