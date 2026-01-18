---
description: Architectural reference for BuilderExpressionStaticStringArray
---

# BuilderExpressionStaticStringArray

**Package:** com.hypixel.hytale.server.npc.asset.builder.expression
**Type:** Transient

## Definition
```java
// Signature
public class BuilderExpressionStaticStringArray extends BuilderExpression {
```

## Architecture & Concepts
The BuilderExpressionStaticStringArray is a foundational component of the server-side NPC expression engine. It represents a *compile-time constant* or a *literal value* of a string array. Architecturally, it serves as a terminal node within an Abstract Syntax Tree (AST) of expressions.

Unlike dynamic expressions which compute their value at runtime using an ExecutionContext, this class holds a predetermined, immutable array. Its primary role is to inject static, predefined data into the expression evaluation pipeline. This is critical for defining fixed lists, configuration sets, or default parameters within NPC behavior scripts without incurring the overhead of dynamic evaluation.

The class adheres to an immutable value object pattern. Once an instance is created, its internal state cannot be altered, which guarantees predictable behavior and thread safety.

### Lifecycle & Ownership
- **Creation:** Instances are primarily created via the static factory method fromJSON during the deserialization of NPC asset files. The server's asset loader parses a JSON definition, identifies a JSON array of strings, and invokes this factory to create a corresponding expression object. The special singleton INSTANCE_EMPTY is used to represent empty arrays, optimizing memory usage.
- **Scope:** The lifecycle of a BuilderExpressionStaticStringArray instance is tightly bound to the lifecycle of the NPC asset that defines it. It persists in memory as part of the parsed expression tree for as long as the parent NPC definition is loaded.
- **Destruction:** The object is eligible for garbage collection when the server unloads the associated NPC asset, and all references to it are released.

## Internal State & Concurrency
- **State:** Immutable. The core state is a private final String array, which is initialized at construction and never modified. The class itself is stateless beyond this initial data.
- **Thread Safety:** This class is inherently thread-safe. Its immutability ensures that it can be safely read and evaluated by multiple threads concurrently without locks or synchronization. The evaluation methods simply return a reference to the internal, unchanging array.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | ValueType | O(1) | Returns the constant ValueType.STRING_ARRAY. |
| isStatic() | boolean | O(1) | Always returns true, signifying its value is constant. |
| getStringArray(ExecutionContext) | String[] | O(1) | Returns the internal string array. The ExecutionContext is ignored. |
| addToScope(String, StdScope) | void | O(1) | Declares a new variable in the provided scope with this array as its value. |
| updateScope(StdScope, String, ExecutionContext) | void | O(1) | Changes the value of an existing variable in the provided scope. |
| fromJSON(JsonArray) | BuilderExpressionStaticStringArray | O(N) | Static factory to construct an instance from a GSON JsonArray. N is the number of elements. Returns null if the JSON is malformed. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in game logic. It is an internal component of the asset loading and expression evaluation system. Its primary integration point is the fromJSON factory method, used by asset parsers.

```java
// Example: Inside an asset loader parsing an NPC behavior file
JsonElement element = jsonObject.get("dialogueOptions");
BuilderExpressionStaticStringArray expression = null;

if (element != null && element.isJsonArray()) {
    expression = BuilderExpressionStaticStringArray.fromJSON(element.getAsJsonArray());
}

// The 'expression' object is then stored in a larger behavior tree for later evaluation.
```

### Anti-Patterns (Do NOT do this)
- **Modifying the Returned Array:** The array returned by getStringArray should be treated as read-only. Although Java arrays are mutable, modifying its contents will violate the immutability contract of the expression system and lead to unpredictable, shared state bugs. This is a critical design constraint.
- **Ignoring Null Return from Factory:** The fromJSON method returns null if the input JsonArray contains non-string or non-primitive elements. Callers must handle this null case to prevent NullPointerExceptions during asset loading.
- **Direct Instantiation for Empty Arrays:** Do not use new BuilderExpressionStaticStringArray(new String[0]). Use the provided singleton BuilderExpressionStaticStringArray.INSTANCE_EMPTY to reduce memory allocations.

## Data Pipeline
The class functions as a data source node in the expression evaluation pipeline. It transforms structured data from a persistent format (JSON) into a runtime-ready object that can be queried by the expression engine.

> Flow:
> NPC Behavior JSON File -> GSON Parser -> `JsonArray` -> **BuilderExpressionStaticStringArray.fromJSON()** -> `BuilderExpression` Tree -> Expression Evaluator -> `String[]` value

