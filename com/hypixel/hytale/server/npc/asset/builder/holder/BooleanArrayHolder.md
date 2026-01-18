---
description: Architectural reference for BooleanArrayHolder
---

# BooleanArrayHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient Data Holder

## Definition
```java
// Signature
public class BooleanArrayHolder extends ArrayHolder {
```

## Architecture & Concepts
The BooleanArrayHolder is a specialized, stateful container within the server's NPC asset-building framework. It is not a generic data structure; rather, it serves as a "smart pointer" to a boolean array defined in a JSON configuration file. Its primary responsibility is to encapsulate the logic for parsing, validating, and dynamically evaluating a `boolean[]` value.

This class extends the generic ArrayHolder, inheriting the core infrastructure for handling array-length constraints and managing dynamic expressions. BooleanArrayHolder enhances this base functionality by introducing type-specific validation for boolean arrays via the BooleanArrayValidator.

It represents a single configurable property of an NPC. The value it holds can be one of two kinds:
1.  **Static:** A literal `boolean[]` defined directly in the JSON. This value is parsed, validated once, and cached internally.
2.  **Dynamic:** An expression or script that resolves to a `boolean[]` at runtime. This requires an ExecutionContext to be evaluated, allowing NPC properties to change based on game state.

## Lifecycle & Ownership
-   **Creation:** BooleanArrayHolder instances are created exclusively by asset-building services during the deserialization of NPC definition files. An instance is typically instantiated for each JSON field that represents a configurable boolean array. The `readJSON` methods act as the primary initialization entry points.

-   **Scope:** The lifetime of a BooleanArrayHolder is strictly bound to the asset construction process. It exists to hold intermediate data and validation logic. Once the final, fully-realized NPC asset is built, the holder object itself is no longer needed and becomes eligible for garbage collection.

-   **Destruction:** Cleanup is managed by the Java garbage collector. The class holds no native resources or persistent connections that would require an explicit destruction method.

## Internal State & Concurrency
-   **State:** The object is highly mutable during its configuration phase (within a `readJSON` call) but is effectively immutable afterward. It stores the compiled expression, length constraints, and a reference to a BooleanArrayValidator. For static values, the resolved `boolean[]` is cached after the first access.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded use within the asset loading pipeline. There are no internal locks or synchronization primitives. Passing an instance between threads or calling its methods concurrently will result in undefined behavior. All interactions with a BooleanArrayHolder must be confined to the thread that is building the asset.

## API Surface
The public API is focused on initialization and value retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(...) | void | O(N) | Initializes the holder from a JsonElement. Complexity is proportional to the size of the JSON data and expression. |
| get(ExecutionContext) | boolean[] | O(E) | Resolves and returns the `boolean[]`. For static values, this is O(1) after the first call. For dynamic values, complexity depends on the expression (E). |
| validate(boolean[]) | void | O(M) | Performs internal validation. Complexity is proportional to the array length (M). Throws IllegalStateException on failure. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay logic developers. It is a framework-internal component. An asset builder service uses it to parse a JSON field.

```java
// Within an asset builder class...
// Assume 'fieldJson' is a JsonElement from a config file.

BooleanArrayHolder myFlags = new BooleanArrayHolder();
BuilderParameters params = ...; // Builder context

// Configure the holder from JSON
myFlags.readJSON(
    fieldJson,
    /* minLength */ 1,
    /* maxLength */ 5,
    /* defaultValue */ new boolean[]{true},
    /* validator */ new MyCustomBooleanValidator(),
    /* name */ "npc.behavior.flags",
    params
);

// Later, at runtime, the game engine resolves the value
ExecutionContext context = ...; // Current game state
boolean[] resolvedFlags = myFlags.get(context);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation without Initialization:** Creating an instance with `new BooleanArrayHolder()` without an immediate call to a `readJSON` method leaves the object in an invalid, unusable state. Any subsequent call to `get` will fail.

-   **State Reconfiguration:** Do not attempt to call `readJSON` more than once on the same instance or modify its internal validators after initial setup. This can lead to a corrupted or inconsistent state.

-   **Ignoring ExecutionContext:** For dynamic values, passing a null or incomplete ExecutionContext to `get` will either fail or produce incorrect results, as the expression cannot be evaluated properly.

## Data Pipeline
The BooleanArrayHolder acts as a crucial step in the transformation of raw configuration data into a usable, in-game property.

> Flow:
> JSON Configuration File -> Gson Parser -> `JsonElement` -> Asset Builder -> **BooleanArrayHolder.readJSON()** -> (Internal `Expression` and `Validator` state)
>
> **Runtime:**
> Game Event -> `ExecutionContext` -> **BooleanArrayHolder.get()** -> `boolean[]` -> NPC Behavior System

