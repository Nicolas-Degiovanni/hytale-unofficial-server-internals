---
description: Architectural reference for AssetArrayHolder
---

# AssetArrayHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient

## Definition
```java
// Signature
public class AssetArrayHolder extends StringArrayHolder {
```

## Architecture & Concepts
The AssetArrayHolder is a specialized data container within the server's NPC asset building framework. It extends the more generic StringArrayHolder, inheriting the capability to hold a string array derived from either a static JSON value or a dynamic expression.

Its primary architectural role is to enforce **asset integrity** at the data-definition level. While a StringArrayHolder simply holds strings, the AssetArrayHolder ensures that these strings are valid asset identifiers according to a provided AssetValidator. This moves validation logic out of the runtime NPC systems and into the initial asset parsing and building phase, allowing for earlier error detection.

This class acts as a crucial bridge between raw configuration data (JSON) and the validated, context-aware asset references required by the game engine. The use of an ExecutionContext in its retrieval methods signifies that the final list of assets can be dynamically determined at runtime, enabling complex NPC behaviors where assets might change based on game state or other variables.

### Lifecycle & Ownership
- **Creation:** An AssetArrayHolder is instantiated by a parent builder class, typically a subclass of BuilderBase, during the deserialization of an NPC definition from a JSON file. It is created specifically to manage a single JSON field that represents a list of asset keys.
- **Scope:** The object's lifetime is strictly bound to its parent builder instance. It persists only for the duration of the asset construction process.
- **Destruction:** It is eligible for garbage collection as soon as the top-level builder has finished constructing the final NPC data object and is itself no longer referenced. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The AssetArrayHolder is a **mutable**, stateful object. Its internal configuration, including the AssetValidator and the underlying expression, is set once via the readJSON methods. After this initial setup, its state is effectively read-only for the remainder of its lifecycle. It does not cache the resolved value from the get method; each call re-evaluates the expression.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It contains no internal locking mechanisms and is designed to be confined to the single thread responsible for asset loading and building. The ExecutionContext passed to its methods is a potential source of concurrency hazards if not managed carefully by the calling system.

## API Surface
The public API is focused on initialization during the build phase and value retrieval during the final object construction phase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(...) | void | O(N) | Initializes the holder from a JSON element. Configures the validator and validation rules. N is the number of elements in the source array. |
| get(executionContext) | String[] | O(E + V) | Resolves the final asset list. E is the complexity of the underlying expression; V is the complexity of all validation checks. This is the primary retrieval method. |
| rawGet(executionContext) | String[] | O(E + V) | Resolves the asset list and performs initial validation but skips relation checks. Intended for internal or specialized use. |
| staticValidate() | void | O(V) | Performs compile-time validation if the underlying expression is static. This is an optimization to fail-fast during server startup. |

## Integration Patterns

### Standard Usage
The AssetArrayHolder is not meant to be used directly. It is an internal component of the asset building system. A parent builder class will manage its lifecycle.

```java
// Conceptual example within a hypothetical NPCDataBuilder

// 1. During JSON parsing, the builder creates a holder for the "requiredModels" field.
AssetArrayHolder modelHolder = new AssetArrayHolder();
modelHolder.readJSON(
    npcJson.get("requiredModels"),
    1, // minLength
    10, // maxLength
    AssetValidators.MODEL_VALIDATOR, // The specific validator
    "requiredModels",
    builderParameters
);

// 2. Later, when the final NPC object is being constructed...
String[] validatedModels = modelHolder.get(npcExecutionContext);

// 3. The validated string array is passed to the NPC's runtime component.
npc.setModels(validatedModels);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an AssetArrayHolder with `new` in game logic. Its creation and configuration are exclusively managed by the asset building framework during server initialization.
- **State Mutation After Read:** Do not attempt to call `readJSON` more than once or modify its state after initial parsing. These objects are designed to be configured once.
- **Using rawGet:** Avoid calling `rawGet` unless you specifically intend to bypass the `validateRelations` check performed by the standard `get` method. Using `rawGet` can lead to runtime errors if related assets are not correctly configured.

## Data Pipeline
The AssetArrayHolder is a key stage in the pipeline that transforms raw text configuration into validated, engine-ready asset references.

> Flow:
> NPC JSON File -> GSON Parser -> JsonElement -> **AssetArrayHolder** (Initialization & Validation Rule Setup) -> `get(context)` call -> **String[]** (Validated Asset Keys) -> NPC Runtime System -> AssetManager Load Request

