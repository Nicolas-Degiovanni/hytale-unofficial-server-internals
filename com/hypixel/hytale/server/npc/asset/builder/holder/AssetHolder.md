---
description: Architectural reference for AssetHolder
---

# AssetHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient State Holder

## Definition
```java
// Signature
public class AssetHolder extends StringHolderBase {
```

## Architecture & Concepts
The AssetHolder is a specialized component within the server's NPC asset construction framework. It serves as a strongly-typed container for a string value that represents a reference to a game asset, such as a model path, texture name, or sound event.

Unlike its parent, StringHolderBase, which is a generic container for string expressions, the AssetHolder integrates directly with the engine's validation system. Its primary architectural role is to bridge the gap between raw, untrusted data from JSON configuration files and the validated, type-safe asset references required by the runtime.

During the deserialization of an NPC definition from JSON, an AssetHolder is instantiated for each field that should point to a valid game asset. It encapsulates not just the string value itself (often as a dynamic expression) but also an **AssetValidator** instance. This coupling ensures that any attempt to retrieve the asset string also triggers a validation check, preventing invalid asset paths from propagating into the game state.

## Lifecycle & Ownership
-   **Creation:** AssetHolder instances are created dynamically by a parent builder class, such as a derivative of BuilderBase, during the parsing of a JSON configuration file. The factory methods are the overloaded `readJSON` members, which populate the instance from a JsonElement.
-   **Scope:** The lifecycle of an AssetHolder is ephemeral. It is bound to the scope of the parent builder object responsible for constructing a single, complex asset (e.g., an entire NPC definition). It exists only for the duration of this construction process.
-   **Destruction:** The object is eligible for garbage collection as soon as the parent builder completes its work and its reference is dropped. There is no manual destruction or cleanup required.

## Internal State & Concurrency
-   **State:** The AssetHolder is a mutable, stateful object. Its internal state, including the expression for the asset path and the associated AssetValidator, is configured post-construction via the `readJSON` methods. Once initialized, its state is typically not modified further.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be used exclusively within the single-threaded context of an asset build pipeline. Concurrent calls to `readJSON` or `get` would result in undefined behavior and race conditions.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(json, validator, name, params) | void | O(1) | Initializes the holder from a required JSON value and a validator. |
| readJSON(json, default, validator, name, params) | void | O(1) | Initializes the holder from an optional JSON value, falling back to a default. |
| get(executionContext) | String | O(N) | Resolves the expression and validates the asset path. Complexity depends on the validator. |
| rawGet(executionContext) | String | O(M) | Resolves the expression and performs basic validation, but skips deeper relation checks. |
| validate(executionContext) | void | O(N) | Explicitly triggers the full validation logic by calling get. |
| staticValidate() | void | O(K) | Performs validation if the asset path is a static, known value at load time. |

## Integration Patterns

### Standard Usage
The AssetHolder is intended to be used internally by higher-level builder classes. A builder reads a JSON field, creates an AssetHolder, and configures it with the appropriate validator for that asset type. Later, during the final object construction phase, it calls `get` to retrieve the validated asset string.

```java
// Within a parent builder class during JSON parsing...
JsonElement modelElement = npcJson.get("model");
AssetValidator modelValidator = AssetValidators.forModels();

// Create and initialize the holder
AssetHolder modelHolder = new AssetHolder();
modelHolder.readJSON(modelElement, modelValidator, "model", builderParameters);

// ... later, when building the final NPC object
ExecutionContext context = new ExecutionContext();
String validatedModelPath = modelHolder.get(context);
npc.setModel(validatedModelPath);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation without Initialization:** Creating an instance with `new AssetHolder()` is useless. The object is in an invalid state until one of the `readJSON` methods is called. Calling `get` before initialization will result in a NullPointerException.
-   **State Re-use:** Do not re-purpose an AssetHolder by calling `readJSON` multiple times. These objects are designed to represent a single field from a single source JSON.
-   **Ignoring ExecutionContext:** Passing a null or incomplete ExecutionContext to `get` may cause dynamic expressions to fail evaluation, leading to incorrect asset paths or runtime errors.

## Data Pipeline
The AssetHolder is a critical stage in the data transformation pipeline from raw text configuration to a live, in-game entity.

> Flow:
> Raw JSON File -> GSON Parser (`JsonElement`) -> Parent Builder -> **AssetHolder.readJSON** -> **AssetHolder** (stores expression & validator) -> Runtime call to **AssetHolder.get** -> AssetValidator -> Validated Asset String -> Game Object Field

