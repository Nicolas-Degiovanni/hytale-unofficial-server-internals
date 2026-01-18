---
description: Architectural reference for BuilderBase
---

# BuilderBase

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public abstract class BuilderBase<T> implements Builder<T> {
```

## Architecture & Concepts
The BuilderBase class is the abstract foundation for the server's entire NPC asset deserialization framework. It implements the generic Builder interface and provides a robust, standardized mechanism for parsing JSON configuration files into concrete Java objects. This class is central to the engine's data-driven design for NPC behaviors, acting as the bridge between raw JSON data and compiled, in-memory game objects.

Its core architectural function is to abstract away the boilerplate and error-prone logic of JSON parsing, type coercion, and validation. Subclasses, such as an ActionBuilder or a ConditionBuilder, extend BuilderBase and use its protected API to declaratively define the structure of their corresponding JSON configuration.

A critical and sophisticated feature of BuilderBase is its **dual-mode operational capability**. The same subclass implementation can be used for two distinct purposes:

1.  **Object Deserialization:** The primary mode, where it reads a JsonElement and populates its internal state to eventually build a runtime object of type T.
2.  **Schema & Descriptor Generation:** A meta-programming mode, invoked by server tooling. In this mode (identified by `isCreatingSchema()` or `isCreatingDescriptor()`), the calls to `require...` and `get...` methods do not parse data but instead contribute to building a formal JSON Schema or a BuilderDescriptor. This allows the framework to be self-documenting and enables static analysis and validation of asset files.

This class is heavily reliant on a collection of helper objects provided during its creation, such as `BuilderManager`, `BuilderValidationHelper`, and `BuilderParameters`. It is not a standalone utility and cannot function outside the context of the asset loading pipeline.

## Lifecycle & Ownership
- **Creation:** A concrete subclass of BuilderBase is instantiated by the `BuilderManager` when it needs to process a specific JSON object. The manager calls the `readConfig` method, passing in the necessary context and the JSON data to be parsed.
- **Scope:** The lifetime of a BuilderBase instance is extremely short and tied to a single parsing operation. It is a transient object that exists only for the duration of the `readConfig` call chain and the subsequent `build` call.
- **Destruction:** The instance becomes eligible for garbage collection immediately after the final object T is constructed and returned. The `BuilderManager` does not retain any long-term references to builder instances. The `postReadConfig` method explicitly nullifies internal collections like `queriedKeys` to aid in cleanup.

## Internal State & Concurrency
- **State:** BuilderBase is fundamentally a stateful class. Its primary purpose is to accumulate state from a JSON source into its member fields and various `ValueHolder` objects. This state is mutable throughout the `readConfig` process. It caches contextual helpers and tracks which JSON keys have been processed to detect unknown attributes.

- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed for synchronous, single-threaded execution within the asset loading pipeline. Its mutable internal state (e.g., `queriedKeys`, `readErrors`, various helpers) would lead to race conditions and unpredictable behavior if accessed concurrently.

    **WARNING:** All interactions with a BuilderBase instance must be confined to the thread that initiated the parsing operation.

## API Surface
The public API is minimal, intended for use by the `BuilderManager`. The primary interaction surface for developers is the extensive protected API used by subclasses to define their configuration structure. The following table summarizes the conceptual groups of protected methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(...) | void | O(N) | The main entry point for parsing. Orchestrates the pre-read, main read, and post-read phases. N is the number of keys in the JSON object. |
| require[Type](...) | void | O(1) | Parses a required JSON attribute of a given type (e.g., requireString, requireInt, requireAsset). Throws an exception if the attribute is missing. |
| get[Type](...) | boolean | O(1) | Parses an optional JSON attribute of a given type (e.g., getString, getDouble). Uses a provided default value if the attribute is absent. Returns true if a value was present in the JSON. |
| expect[Type](...) | [Type] | O(1) | Low-level utility to extract and coerce a JsonElement to a specific primitive type. Throws if the type is incorrect. |
| validate[Rule](...) | void | O(k) | Adds a validation rule to be checked either immediately or at a later stage (e.g., validateIntRelation, validateOnePresent). k depends on the complexity of the rule. |
| addError(...) | void | O(1) | Records a parsing or validation error, associating it with the current file context. |

## Integration Patterns

### Standard Usage
A developer never instantiates BuilderBase directly. Instead, they create a concrete builder for a specific asset type, extend BuilderBase, and implement the `readConfig` and `build` methods. Inside `readConfig`, they use the protected `require...` and `get...` methods to parse the JSON data.

```java
// Example of a concrete subclass using BuilderBase's API
public class MyActionBuilder extends BuilderBase<MyAction> {
    private StringHolder targetEntity;
    private IntHolder duration;

    @Override
    public Builder<MyAction> readConfig(JsonElement data) {
        // Use protected methods from BuilderBase to define the JSON contract
        this.targetEntity = new StringHolder();
        this.duration = new IntHolder();

        requireString(data, "Target", this.targetEntity, StringValidator.NOT_EMPTY, ...);
        getInt(data, "Duration", this.duration, 20, IntValidator.POSITIVE, ...);

        return this;
    }

    @Override
    public MyAction build(BuilderSupport support) {
        // Use the parsed data to construct the final object
        return new MyAction(targetEntity, duration);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new MyBuilder()`. The `BuilderManager` is responsible for the entire lifecycle. Direct instantiation will fail because critical context helpers (like `BuilderValidationHelper`) will be null.
- **State Re-use:** Do not attempt to re-use a builder instance to parse a second JSON object. The internal state, particularly `queriedKeys` and error lists, is not designed for reset.
- **Concurrent Access:** Do not pass a builder instance to another thread. The internal state is not protected by locks and will become corrupted.
- **Ignoring Context:** Do not call parsing methods without the full context provided by the `readConfig` entry point. Methods rely on members like `builderParameters` and `validationHelper` which are set up in `preReadConfig`.

## Data Pipeline
BuilderBase is a key processing stage in the NPC asset loading pipeline. It transforms structured text data into validated, memory-efficient runtime objects.

> Flow:
> JSON File -> Gson Parser -> `JsonElement` -> `BuilderManager` -> **`BuilderBase` Subclass** -> Validation & Linking -> Final Game Object (e.g., Action, Condition) -> NPC Runtime System

