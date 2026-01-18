---
description: Architectural reference for BuilderExpressionStaticNumber
---

# BuilderExpressionStaticNumber

**Package:** com.hypixel.hytale.server.npc.asset.builder.expression
**Type:** Transient / Value Object

## Definition
```java
// Signature
public class BuilderExpressionStaticNumber extends BuilderExpression {
```

## Architecture & Concepts
The BuilderExpressionStaticNumber is a foundational component of the server-side NPC expression engine. It serves as a leaf node within an expression's Abstract Syntax Tree (AST), representing a constant, literal numeric value.

Its primary architectural role is to encapsulate a value that is known at asset-build time and does not change during runtime evaluation. The `isStatic` method, which always returns true, signals this immutability to the expression evaluator. This allows the engine to perform potential optimizations, such as pre-calculating parts of an expression tree that consist entirely of static values.

This class is one of the simplest implementations of the BuilderExpression contract, providing a terminal value that more complex expressions (like arithmetic or conditional operators) can operate on. It is the in-memory representation of a number written directly into an NPC asset definition file.

## Lifecycle & Ownership
- **Creation:** Instances are created by the NPC asset parsing system. When the parser encounters a literal number within an expression string or structure (e.g., in a JSON file), it instantiates a BuilderExpressionStaticNumber to represent that value in the object graph.
- **Scope:** The object's lifetime is bound to the containing expression tree. It persists as long as the parent NPC asset definition is loaded in memory.
- **Destruction:** As a simple value object with no external resource handles, it is garbage collected when the root of its expression tree is no longer referenced. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** Immutable. The internal `number` field is declared as `private final` and is initialized exclusively through the constructor. Once created, an instance's value can never be changed.
- **Thread Safety:** This class is inherently thread-safe. Its immutability guarantees that it can be safely read, evaluated, and shared across multiple threads without any risk of data corruption or race conditions. No synchronization mechanisms are necessary.

## API Surface
The public API is designed for interaction with the expression evaluation engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | ValueType | O(1) | Returns the constant ValueType.NUMBER, identifying its type to the engine. |
| isStatic() | boolean | O(1) | Returns the constant true, signaling that its value is context-independent. |
| getNumber(ExecutionContext) | double | O(1) | Returns the stored number. **Warning:** The ExecutionContext argument is ignored. |
| addToScope(String, StdScope) | void | O(1) | Adds its static number as a new variable to the provided scope. |
| updateScope(StdScope, String, ExecutionContext) | void | O(1) | Updates an existing variable in the provided scope with its static number. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game-logic developers. It is an internal component used by the expression engine itself. The typical interaction occurs when a more complex expression evaluates its children.

```java
// Example from within a hypothetical BuilderExpressionAdd class
BuilderExpression left = new BuilderExpressionStaticNumber(10.0);
BuilderExpression right = new BuilderExpressionStaticNumber(5.0);

// The engine evaluates the expression, which in turn evaluates its children
double result = left.getNumber(context) + right.getNumber(context); // result is 15.0
```

### Anti-Patterns (Do NOT do this)
- **External Instantiation:** Game feature code should never create instances using `new BuilderExpressionStaticNumber()`. NPC behavior should be defined in asset files, which are then parsed by the appropriate system.
- **Expecting Dynamic Behavior:** Do not attempt to use this class for a value that needs to change based on game state. Its value is fixed at creation. Attempting to get a dynamic result from it indicates a misunderstanding of its purpose.

## Data Pipeline
The flow of data from configuration to a usable value is linear and unidirectional. This component exists at the midpoint of this pipeline, acting as the in-memory representation of a configured value.

> Flow:
> NPC Asset File (e.g., JSON) -> Asset Parser -> **BuilderExpressionStaticNumber Instance** -> Expression Evaluator -> Final `double` Result

