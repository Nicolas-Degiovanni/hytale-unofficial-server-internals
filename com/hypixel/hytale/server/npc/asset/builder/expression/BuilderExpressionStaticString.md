---
description: Architectural reference for BuilderExpressionStaticString
---

# BuilderExpressionStaticString

**Package:** com.hypixel.hytale.server.npc.asset.builder.expression
**Type:** Transient / Value Object

## Definition
```java
// Signature
public class BuilderExpressionStaticString extends BuilderExpression {
```

## Architecture & Concepts
The BuilderExpressionStaticString is a foundational component of the server-side NPC expression engine. It represents a constant, immutable string value known at asset-load time. This class serves as a leaf node within an expression's Abstract Syntax Tree (AST), providing a literal value that requires no runtime evaluation or context.

Its primary role is to encapsulate a static string parsed from an NPC asset definition. Unlike dynamic expressions that might depend on game state (e.g., player health, time of day), this class guarantees that its output is always the same, making it highly predictable and performant. It is the simplest concrete implementation of the BuilderExpression contract.

## Lifecycle & Ownership
- **Creation:** Instances are created by the NPC asset expression parser. When the parser encounters a string literal within an expression (e.g., a quoted string in a configuration file), it instantiates this class to represent that value in the in-memory expression tree.
- **Scope:** The lifetime of a BuilderExpressionStaticString instance is bound to the lifecycle of the parent expression tree it belongs to. It persists as long as the parsed NPC asset is loaded in memory.
- **Destruction:** The object is eligible for garbage collection once the containing expression tree is dereferenced, typically when an NPC asset is unloaded or the server shuts down.

## Internal State & Concurrency
- **State:** This class is **immutable**. Its internal state consists of a single final string, which is set at construction and can never be changed. It performs no caching and holds no references to mutable game state.
- **Thread Safety:** BuilderExpressionStaticString is **inherently thread-safe**. Due to its immutable nature, instances can be safely shared and accessed by multiple threads without synchronization. This is critical for systems that may evaluate NPC logic concurrently.

## API Surface
The public contract is minimal, focusing on value retrieval and scope manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | ValueType | O(1) | Returns the constant ValueType.STRING. |
| isStatic() | boolean | O(1) | Always returns true, signaling that its value is constant and known ahead of execution. |
| getString(executionContext) | String | O(1) | Returns the encapsulated string. The ExecutionContext argument is ignored. |
| addToScope(name, scope) | void | O(1) | Binds the static string value to a variable name within the provided StdScope. |
| updateScope(scope, name, executionContext) | void | O(1) | Changes an existing variable in the provided StdScope to the static string value. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic developers. It is an internal component of the expression evaluation system. The parser constructs it, and the evaluator consumes it.

```java
// Example: Internal usage by an expression evaluator
// Assume 'expression' is an instance of BuilderExpressionStaticString
ExecutionContext context = ...;
String dialogueLine = expression.getString(context);
// dialogueLine now holds the static string from the asset file
```

### Anti-Patterns (Do NOT do this)
- **Manual Construction:** Avoid creating instances of this class manually in game systems. Expressions should be defined in asset files and processed by the engine's parser. Manual creation bypasses the intended asset pipeline.
- **Dynamic Data Representation:** Do not attempt to use this class to represent a string that changes based on game state. Its design is exclusively for compile-time constants. Use a dynamic expression type for values that must be resolved at runtime.

## Data Pipeline
This component acts as a data source node within the expression evaluation pipeline. It introduces a static, pre-defined value into the system.

> Flow:
> NPC Asset File -> Expression Parser -> **BuilderExpressionStaticString Instance** -> Expression Evaluator -> String Value -> Consumer System (e.g., Dialogue Engine, Behavior Controller)

