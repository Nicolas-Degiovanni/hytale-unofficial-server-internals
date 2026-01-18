---
description: Architectural reference for BuilderObjectArrayHelper
---

# BuilderObjectArrayHelper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient Component

## Definition
```java
// Signature
public abstract class BuilderObjectArrayHelper<T, U> extends BuilderObjectHelper<T> {
```

## Architecture & Concepts

The BuilderObjectArrayHelper is a specialized component within the server's NPC asset loading framework. It is not a standalone service but rather a stateful, transient object responsible for deserializing and validating a JSON array from an NPC configuration file.

Its primary role is to act as a **composite parser** for a collection of related sub-objects. Where a standard BuilderObjectHelper might handle a single JSON object, the BuilderObjectArrayHelper is designed explicitly to process an array of objects. It transforms a `JsonArray` into an internal array of BuilderObjectReferenceHelper instances, each representing a single element from the source array.

This class is a key enabler of the framework's hierarchical, recursive-descent parsing strategy. It is instantiated by a parent builder when it encounters a property that is expected to be an array. The generic type `T` represents the object type being constructed by the parent, while `U` represents the type of the objects contained within the array that this helper manages.

## Lifecycle & Ownership

-   **Creation:** An instance of BuilderObjectArrayHelper is never created directly by application code. It is instantiated by the core `BuilderManager` or a parent builder during the recursive `readConfig` phase of asset loading. Its creation is triggered by the presence of a corresponding JSON key whose value is a JSON array.

-   **Scope:** The object's lifetime is strictly scoped to a single asset loading and validation operation. It serves as an intermediate container for the parsed, but not yet finalized, configuration data.

-   **Destruction:** The object holds no persistent resources and has no explicit destruction method. It becomes eligible for garbage collection as soon as the top-level asset loading process completes and the final, in-memory game objects have been constructed from its data.

## Internal State & Concurrency

-   **State:** The class is highly mutable. Its primary internal state is the `builders` array of BuilderObjectReferenceHelper. This field is `null` before `readConfig` is called and is populated during the call. Its state is therefore path-dependent on the content of the JSON being parsed.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be used exclusively within the single-threaded context of an asset loading operation. The `readConfig` method directly mutates the internal `builders` array. Concurrent access would result in race conditions, data corruption, and unpredictable behavior. All interactions with this class must be confined to the thread performing the asset load.

## API Surface

The public API is designed for internal framework consumption during the two main phases of asset loading: parsing and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(...) | void | O(N) | Populates the helper by parsing a `JsonElement`. Throws `IllegalArgumentException` if the element is not a JSON array. N is the number of elements in the array. |
| validate(...) | boolean | O(N) | Recursively validates all child builders contained within this array. Aggregates validation results and errors. |
| isPresent() | boolean | O(1) | Returns true if the corresponding key was present in the JSON, even if the value was an empty array `[]`. |
| hasNoElements() | boolean | O(1) | Returns true if the key was missing, the value was `null`, or the value was an empty array `[]`. |

## Integration Patterns

### Standard Usage

This class is an internal implementation detail of the asset framework and should not be invoked directly. A game developer interacts with it implicitly by defining a JSON array in an NPC configuration file. The framework then uses BuilderObjectArrayHelper under the hood.

*Example: A developer defines an array of `abilities` in an NPC file.*
```json
// in my_monster.json
{
  "name": "Goblin Shaman",
  "abilities": [
    { "ref": "hytale:fireball" },
    { "ref": "hytale:heal_self" }
  ]
}
```

The `BuilderManager`, while parsing `my_monster.json`, would detect the `abilities` key and delegate its processing to a BuilderObjectArrayHelper instance. This helper would then parse the two-element array, creating two child `BuilderObjectReferenceHelper` instances.

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new BuilderObjectArrayHelper()`. The object is tightly coupled to the lifecycle and context (`BuilderContext`, `BuilderManager`) of the asset loading pipeline. Manual creation will result in a non-functional, isolated object.

-   **State Assumption:** Do not attempt to access the internal `builders` array without first checking the object's state. Accessing the array after `readConfig` with a `null` or non-array JSON value will result in a `NullPointerException`. Always use `isPresent()` or `hasNoElements()` to check state first.

-   **Concurrent Modification:** Do not share an instance of this helper across threads. The asset loading process is strictly single-threaded, and this component's design relies on that guarantee.

## Data Pipeline

The BuilderObjectArrayHelper functions as a specific stage in the broader NPC asset deserialization and validation pipeline. It takes a raw JSON structure and converts it into a validated, intermediate object graph.

> Flow:
> NPC JSON File on Disk -> Gson Parser -> `JsonArray` -> `BuilderManager` -> **BuilderObjectArrayHelper.readConfig** -> Populated `BuilderObjectReferenceHelper[]` -> **BuilderObjectArrayHelper.validate** -> Validated Asset Fragment -> Final Game Object Construction

