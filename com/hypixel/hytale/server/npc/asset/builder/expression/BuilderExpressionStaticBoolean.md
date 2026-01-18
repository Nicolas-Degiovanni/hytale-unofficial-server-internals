---
description: Architectural reference for BuilderExpressionStaticBoolean
---

# BuilderExpressionStaticBoolean

**Package:** com.hypixel.hytale.server.npc.asset.builder.expression
**Type:** Transient

## Definition
```java
// Signature
public class BuilderExpressionStaticBoolean extends BuilderExpression {
```

## Architecture & Concepts
The BuilderExpressionStaticBoolean is a fundamental component of the server-side NPC expression evaluation system. It serves as a leaf node within a larger expression tree, representing a compile-time, constant boolean value—either *true* or *false*.

Its primary architectural role is to provide an immutable, context-free value. Unlike dynamic expressions that must be evaluated against a runtime ExecutionContext, this class's value is fixed upon instantiation. The system leverages the `isStatic()` method, which returns `true`, to perform significant optimizations. The expression evaluator can short-circuit logic or pre-calculate outcomes involving this node without needing to consult the current game state.

This class is a concrete implementation of the Expression pattern, specifically for literal boolean values found in NPC behavior definitions or scripts.

### Lifecycle & Ownership
- **Creation:** An instance is created by an expression parser or a higher-level asset builder when it encounters a literal boolean value (e.g., `true`) during the deserialization of an NPC behavior asset. It is never created directly by game logic systems.
- **Scope:** The object's lifetime is bound to the expression tree it is a part of. It persists as long as the compiled NPC behavior is held in memory.
- **Destruction:** The object is marked for garbage collection when the parent expression tree is discarded, for instance, when an NPC type is unloaded or a server shuts down.

## Internal State & Concurrency
- **State:** Immutable. The internal `bool` field is declared `final` and is set exclusively by the constructor. Once created, an instance of BuilderExpressionStaticBoolean represents a single, unchangeable boolean value for its entire lifetime.
- **Thread Safety:** This class is inherently thread-safe. Its immutability guarantees that it can be safely read, evaluated, and shared across multiple threads—such as parallel AI behavior ticks—without locks or any other synchronization primitives.

## API Surface
The public API is designed for interaction with the expression evaluation engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | ValueType | O(1) | Returns the constant ValueType.BOOLEAN. |
| isStatic() | boolean | O(1) | Returns `true`, signaling to the engine that its value is constant. |
| getBoolean(ExecutionContext) | boolean | O(1) | Returns the stored boolean value. **Warning:** The ExecutionContext parameter is ignored. |
| addToScope(String, StdScope) | void | O(1) | Adds its static boolean value to a provided scope under the given name. |
| updateScope(StdScope, String, ExecutionContext) | void | O(1) | Updates an existing variable in a scope to its static boolean value. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers writing game logic. It is an internal building block used by the NPC asset pipeline. The system uses it to construct an executable representation of a static boolean.

```java
// The expression engine uses this class to represent a literal value.
// For example, when parsing a rule like "isHostile = true".

StdScope npcRuntimeScope = new StdScope();
BuilderExpression staticTrueValue = new BuilderExpressionStaticBoolean(true);

// The engine populates the initial state of an NPC from the expression tree.
staticTrueValue.addToScope("isHostile", npcRuntimeScope);

// Later, the value can be retrieved during evaluation.
boolean isHostile = npcRuntimeScope.getBoolean("isHostile"); // Returns true
```

### Anti-Patterns (Do NOT do this)
- **Representing Dynamic State:** Do not use this class for a value that depends on game state (e.g., "is player nearby?"). Its value is static and ignores the ExecutionContext. Using it for dynamic data will lead to incorrect and unresponsive NPC behavior. A dynamic expression implementation must be used instead.
- **Attempting to Modify State:** The object is immutable. Any attempt to change its internal value via reflection or other means violates its design contract and will lead to unpredictable behavior across the entire expression evaluation system.

## Data Pipeline
BuilderExpressionStaticBoolean acts as a data source within the expression evaluation pipeline. It does not process incoming data; it originates a static value that flows into other parts of the system.

> **Flow:**
> NPC Behavior Asset (JSON) -> Asset Deserializer -> **BuilderExpressionStaticBoolean Instance** -> In-Memory Expression Tree -> Expression Evaluator -> NPC Runtime Scope (StdScope) -> Behavior Tree Execution

