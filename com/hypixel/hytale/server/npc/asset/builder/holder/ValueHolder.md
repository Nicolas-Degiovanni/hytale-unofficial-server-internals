---
description: Architectural reference for ValueHolder
---

# ValueHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class ValueHolder {
```

## Architecture & Concepts
The ValueHolder is an abstract base class that serves as a foundational component within the server's NPC asset building framework. Its primary architectural purpose is to encapsulate a single value defined within a JSON asset file.

Crucially, a ValueHolder does not typically store a concrete, final value. Instead, it holds a **BuilderExpression**, which is an unevaluated representation of that value. This design decouples the asset definition (parsing) from its runtime evaluation. This allows NPC properties to be defined as static constants (e.g., `10`), variables (e.g., `world.difficulty`), or complex formulas (e.g., `5 + level * 2`), which are only resolved when needed within a specific ExecutionContext.

This class forms the bridge between the raw JSON asset data and the dynamic, context-aware expression evaluation system. Concrete implementations, such as an IntegerHolder or a BooleanHolder, extend this class to provide type-specific validation and behavior.

### Lifecycle & Ownership
- **Creation:** A ValueHolder is never instantiated directly. Concrete subclasses are created by higher-level asset builders (e.g., an NpcDefinitionBuilder) during the deserialization of a JSON asset. The `readJSON` method is the primary mechanism for initializing its state from the source data.
- **Scope:** The lifecycle of a ValueHolder instance is tightly bound to the lifecycle of the parent asset that defines it. It persists in memory as long as its containing asset (e.g., an NPC definition) is loaded.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup once the parent asset is unloaded and no longer referenced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The state of a ValueHolder is mutable, primarily during the asset loading phase. The `name` and `expression` fields are populated by the `readJSON` method after the object is constructed. Once the asset loading process is complete, the state of a ValueHolder should be considered effectively immutable.

- **Thread Safety:** **This class is not thread-safe.** All initialization and state modification via methods like `readJSON` and `setName` must be performed on the thread responsible for asset loading. Concurrent access or modification will lead to race conditions and undefined behavior. The contained BuilderExpression may have its own concurrency guarantees for evaluation, but the holder object itself provides none.

## API Surface
The public contract is designed for use by asset builders and the expression system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validate(ExecutionContext) | abstract void | O(N) | Validates the contained expression against a given context. Throws if invalid. N is the complexity of the expression. |
| readJSON(...) | protected void | O(M) | Populates the holder's state from a JsonElement. This is the core initialization path. M is the size of the JSON input. |
| isStatic() | boolean | O(1) | Returns true if the contained expression is a static constant that can be evaluated at load time without a context. |
| getExpressionString() | String | O(1) | Returns the raw, unevaluated string representation of the expression. |

## Integration Patterns

### Standard Usage
A ValueHolder is intended to be used by a parent builder class that parses a larger JSON structure. The parent is responsible for creating the correct subclass and invoking `readJSON`.

```java
// Example within a hypothetical parent builder
public class NpcStatsBuilder {
    private IntegerHolder health; // Concrete subclass of ValueHolder

    public void parse(JsonElement statsJson, BuilderParameters params) {
        JsonObject statsObject = statsJson.getAsJsonObject();

        // Create the holder and delegate parsing
        this.health = new IntegerHolder(); // Or retrieved from a pool
        this.health.readJSON(
            statsObject.get("health"),
            "health",
            params
        );
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Load:** Do not call `setName` or `readJSON` on a ValueHolder after the initial asset loading phase is complete. This can corrupt the state of a shared asset definition.
- **Ignoring Validation:** Skipping the `validate` step can result in difficult-to-diagnose runtime exceptions when the expression is finally evaluated. All expressions should be validated after an asset is fully loaded.
- **Incorrect Subclass:** Using the wrong type of holder (e.g., an IntegerHolder for a string value) will cause parsing or validation failures.

## Data Pipeline
The ValueHolder is a key stage in the pipeline that transforms a JSON definition on disk into a usable, dynamic value at runtime.

> Flow:
> JSON Asset File -> Gson Parser -> Parent Asset Builder -> **ValueHolder.readJSON** -> In-Memory ValueHolder Instance -> Runtime Evaluation with ExecutionContext -> Final Typed Value

