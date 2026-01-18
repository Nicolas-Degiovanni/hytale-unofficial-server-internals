---
description: Architectural reference for BooleanHolder
---

# BooleanHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient Component

## Definition
```java
// Signature
public class BooleanHolder extends ValueHolder {
```

## Architecture & Concepts
The BooleanHolder is a specialized component within the server-side NPC Asset Builder framework. It is not a simple container for a primitive boolean. Instead, it represents a configurable boolean value that is defined in a JSON asset and resolved dynamically at runtime.

Its core architectural function is to decouple the *definition* of a property from its *evaluation*. The value is stored internally as a BuilderExpression, which is evaluated against a runtime ExecutionContext. This allows for complex, context-sensitive NPC configurations, where a boolean property might depend on the current game state or other NPC attributes.

A critical feature is the **relation validation** system. BooleanHolder acts as a node in a dependency graph of configuration properties. Other components can register validation logic via `addRelationValidator`. When the boolean value is accessed through the primary `get` method, all registered validators are executed, ensuring that related NPC properties are consistent with this boolean's resolved value.

## Lifecycle & Ownership
- **Creation:** A BooleanHolder is instantiated exclusively by a parent builder class during the deserialization of an NPC JSON asset file. It is never intended to be created directly by application logic.

- **Scope:** The lifecycle of a BooleanHolder is strictly bound to its parent builder instance. It persists only for the duration of the asset loading, parsing, and validation process.

- **Destruction:** The object is marked for garbage collection once the parent builder completes its work and the resulting NPC asset definition is finalized. It does not manage any native resources and has no explicit cleanup or `close` method.

## Internal State & Concurrency
- **State:** The internal state is **mutable** during the initial asset parsing phase. The `readJSON` method sets the underlying expression, and `addRelationValidator` populates the list of validators. After this configuration stage, the object is treated as effectively immutable.

- **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access during the asset loading sequence. The internal `relationValidators` list is not synchronized. Concurrent modification or access from multiple threads will result in race conditions and unpredictable behavior.

## API Surface
The public API is designed for a two-phase process: configuration followed by evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(json, name, params) | void | O(1) | Configures the holder from a required JSON element. |
| readJSON(json, default, name, params) | void | O(1) | Configures the holder from an optional JSON element, using a default if absent. |
| get(executionContext) | boolean | O(N) | Resolves the expression and executes all N registered validators. This is the primary access method. |
| rawGet(executionContext) | boolean | O(1) | Resolves the expression *without* executing validators. **Warning:** Use is heavily discouraged. |
| addRelationValidator(validator) | void | O(1) | Registers a validation callback that depends on this holder's value. |

## Integration Patterns

### Standard Usage
The BooleanHolder is managed entirely by a parent builder. The builder populates it from JSON and then, during a final validation pass, its value is retrieved to ensure asset integrity.

```java
// Within a parent builder class during asset parsing...

// 1. Create and configure the holder from a JSON source
BooleanHolder canFly = new BooleanHolder();
canFly.readJSON(jsonObject.get("canFly"), false, "canFly", builderParameters);

// 2. Another component registers a dependency
someOtherHolder.addFlySpeedValidator(canFly);

// 3. During a final validation step, the value is resolved
// This triggers all registered validators in other components.
boolean isFlyingEnabled = canFly.get(validationContext);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create a BooleanHolder using `new BooleanHolder()` in general application code. It is an internal component of the asset building system and must be managed by a parent builder.

- **Bypassing Validation:** Avoid calling `rawGet`. This method bypasses the entire relation validation system, potentially leading to the creation of NPCs with inconsistent or corrupt state. It exists for highly specific internal framework use cases only.

- **Retaining References:** Do not store a reference to a BooleanHolder outside the scope of the asset building process. Doing so constitutes a memory leak, as it prevents the entire builder object graph from being garbage collected.

## Data Pipeline
The BooleanHolder functions within a configuration and validation pipeline, not a real-time data stream.

> Flow:
> JSON Asset File -> GSON Parser -> Parent Builder -> **BooleanHolder.readJSON** -> Stored BuilderExpression -> **BooleanHolder.get(context)** -> Resolved `boolean` & Execution of Relation Validators
---

