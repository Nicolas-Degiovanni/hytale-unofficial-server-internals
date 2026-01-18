---
description: Architectural reference for BuilderExpressionStaticNumberArray
---

# BuilderExpressionStaticNumberArray

**Package:** com.hypixel.hytale.server.npc.asset.builder.expression
**Type:** Transient Value Object

## Definition
```java
// Signature
public class BuilderExpressionStaticNumberArray extends BuilderExpression {
```

## Architecture & Concepts
The **BuilderExpressionStaticNumberArray** is a concrete implementation of the **BuilderExpression** contract. Its role within the server's NPC behavior system is to represent a *compile-time constant*: an array of numbers whose values are known when assets are loaded and do not change during gameplay.

The "Static" in its name is critical; it signifies that the value is independent of the runtime **ExecutionContext**. This makes it a highly efficient node in the expression tree, as its value can be resolved without invoking game logic or querying world state.

This class acts as a bridge between the asset serialization layer (JSON) and the live expression evaluation engine. It encapsulates a literal numeric array defined within an NPC asset file, providing a standardized interface for the expression evaluator to consume. It stands in contrast to dynamic expressions which might compute an array's contents based on in-game events or entity state.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created via the static factory method **fromJSON** during the server's asset parsing phase. When an NPC behavior asset is loaded from disk, this factory is invoked to deserialize any literal JSON arrays of numbers into an in-memory object representation. For empty arrays, the shared singleton **INSTANCE_EMPTY** is used to prevent unnecessary object allocation.
- **Scope:** The lifetime of a **BuilderExpressionStaticNumberArray** instance is tightly coupled to its parent NPC asset definition. It persists in memory for as long as the server holds the parsed NPC asset.
- **Destruction:** The object is eligible for garbage collection when the corresponding NPC asset is unloaded. This typically occurs during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** The primary state, a double array named **numberArray**, is **immutable** due to being a final field. However, the class implements a lazy-initialization pattern for an integer-based representation of this array. The **cachedIntArray** field is mutable, being populated on the first call to **getIntegerArray**. This is a performance optimization to avoid repeated type conversions.

- **Thread Safety:** This class is **not thread-safe**. The lazy initialization logic within **createCacheIfAbsent** presents a race condition. If multiple threads call **getIntegerArray** concurrently on a newly created instance, the conversion from double to int may be performed multiple times. While the operation is idempotent (the result will be the same), this is inefficient and violates thread-safety principles. The design implicitly assumes that asset parsing and expression tree evaluation occur within a single-threaded context.

## API Surface
The public API is minimal, focusing on value retrieval and interaction with the expression evaluation scope.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromJSON(JsonArray) | BuilderExpressionStaticNumberArray | O(N) | **Static Factory.** Parses a JSON array into a new instance. Returns null on validation failure. |
| getNumberArray(ExecutionContext) | double[] | O(1) | Returns a direct reference to the internal double array. Ignores the execution context. |
| getIntegerArray(ExecutionContext) | int[] | O(N) / O(1) | Returns an integer representation. O(N) on first call due to conversion and caching; O(1) on subsequent calls. |
| addToScope(String, StdScope) | void | O(1) | Binds the internal array to a variable name within the provided expression scope. |
| updateScope(StdScope, String, ExecutionContext) | void | O(1) | Changes the value of an existing variable in the scope to this object's internal array. |

## Integration Patterns

### Standard Usage
This class is an internal component of the asset pipeline and expression engine. It is not intended for direct manipulation by gameplay logic developers. Its usage is orchestrated by higher-level systems.

```java
// Example: Simplified asset loader logic
JsonArray sourceData = parseJsonFile("npc_behavior.json");
BuilderExpression expression = BuilderExpressionStaticNumberArray.fromJSON(sourceData);

// Later, the expression engine evaluates it
ExecutionContext context = ...;
double[] values = expression.getNumberArray(context);
// Use values in game logic...
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid `new BuilderExpressionStaticNumberArray()`. The canonical entry point is the **fromJSON** factory, which handles validation. For empty arrays, the **INSTANCE_EMPTY** singleton must be used to conserve memory.
- **Concurrent Access:** Do not call **getIntegerArray** from multiple threads on the same instance without external locking. The internal cache initialization is not thread-safe.
- **Returned Array Modification:** The **getNumberArray** and **getIntegerArray** methods return a direct reference to the internal state arrays. Modifying these arrays externally will corrupt the object's state and violate the principle of static, immutable expressions. This can lead to severe and difficult-to-diagnose bugs.

## Data Pipeline
The flow of data for this component is linear, originating from a raw asset file and terminating as a primitive array used by the game's logic.

> Flow:
> NPC Asset JSON File -> Server Asset Parser (Gson) -> **BuilderExpressionStaticNumberArray.fromJSON** -> In-Memory Expression Tree Node -> Expression Evaluator -> **getNumberArray()** -> Primitive double[] for NPC Logic

