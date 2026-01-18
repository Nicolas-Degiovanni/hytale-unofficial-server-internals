---
description: Architectural reference for StringHolder
---

# StringHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient State Holder

## Definition
```java
// Signature
public class StringHolder extends StringHolderBase {
```

## Architecture & Concepts
The StringHolder is a fundamental component within the server-side NPC asset building framework. Its purpose is to represent a string value that is sourced from a JSON configuration file, while encapsulating critical logic for dynamic evaluation and runtime validation.

This class is not a simple data container. It acts as a deferred computation and validation node in an asset construction graph. It can represent either a static, compile-time string or a dynamic expression that is resolved at runtime using an ExecutionContext. This distinction is critical for performance and flexibility; static values are validated once at load time, while dynamic values are validated on each access.

The StringHolder integrates a pluggable StringValidator, allowing asset definitions to enforce arbitrary business rules (e.g., format constraints, length limits, or registry lookups) on the string values they consume. It is a key enabler for creating robust, data-driven, and error-resistant NPC definitions.

## Lifecycle & Ownership
- **Creation:** A StringHolder is instantiated exclusively by a higher-level builder class during the deserialization of an NPC asset from JSON. The builder calls one of the `readJSON` methods to configure the holder with its source data, name, and validation rules.
- **Scope:** The lifetime of a StringHolder is strictly bound to the asset-building operation. It persists in memory only for the duration required to construct a single, complete NPC asset object.
- **Destruction:** The object is relinquished and becomes eligible for garbage collection as soon as the parent builder completes its task and the final NPC asset is returned. It maintains no static references and is not intended to outlive the build process.

## Internal State & Concurrency
- **State:** The StringHolder is a stateful, mutable object. Its internal state, including the expression and the StringValidator, is configured once during the `readJSON` initialization phase. The class does not cache the resolved string value from the expression; each call to `get` may trigger a re-evaluation.
- **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded use within the asset-building pipeline. The ExecutionContext passed to its methods may contain thread-specific state, and the class itself employs no locking or synchronization. Concurrent access is unsupported and will lead to unpredictable behavior and validation failures.

## API Surface
The public API is designed for a two-phase interaction: initialization followed by value retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(...) | void | O(V) | Initializes the holder from a JsonElement. Configures the internal expression and validator. If the expression is static, validation (cost V) is performed immediately. |
| get(executionContext) | String | O(E + V) | Resolves the expression (cost E) and validates the result (cost V). This is the primary method for retrieving the final string value. Throws IllegalStateException on validation failure. |
| rawGet(executionContext) | String | O(E + V) | Resolves the expression and performs validation only if the expression is dynamic. Bypasses relation validation. **Warning:** Prefer `get` for most use cases. |
| validate(value) | void | O(V) | Directly invokes the configured validator against the provided string. Throws IllegalStateException on failure. |

## Integration Patterns

### Standard Usage
The StringHolder is intended to be used internally by asset builder classes. A builder first initializes the holder from a JSON source and later retrieves the validated value using a runtime context.

```java
// Within a hypothetical NPC asset builder...

// 1. Initialization during JSON parsing
StringHolder modelNameHolder = new StringHolder();
JsonElement modelNameJson = jsonObject.get("model");
modelNameHolder.readJSON(modelNameJson, new ModelRegistryValidator(), "model", builderParams);

// ... other properties are processed ...

// 2. Retrieval during final object construction
ExecutionContext context = new ExecutionContext(targetEntity);
String finalModelName = modelNameHolder.get(context);
npc.setModel(finalModelName);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation without Initialization:** Do not use `new StringHolder()` and then immediately call `get`. The internal expression will be null, causing a NullPointerException. Always initialize via `readJSON`.
- **State Re-use:** Do not reuse a StringHolder instance across the construction of multiple, independent NPC assets. Its lifecycle is tied to a single build operation.
- **Multi-threaded Access:** Do not share a StringHolder or its parent builder across threads. The entire asset-building process is single-threaded by design.
- **Bypassing Validation:** Avoid calling `rawGet` unless you have a specific need to bypass relation validation and understand the consequences. The standard `get` method provides the safest contract.

## Data Pipeline
The StringHolder processes data in a linear flow from raw configuration to a validated, runtime-ready value.

> Flow:
> JSON File -> GSON Parser -> JsonElement -> **StringHolder.readJSON** -> (Internal Expression & Validator) -> **StringHolder.get(context)** -> Validated String -> NPC Asset Field

