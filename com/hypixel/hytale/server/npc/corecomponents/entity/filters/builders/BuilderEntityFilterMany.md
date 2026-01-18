---
description: Architectural reference for BuilderEntityFilterMany
---

# BuilderEntityFilterMany

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters.builders
**Type:** Transient

## Definition
```java
// Signature
public abstract class BuilderEntityFilterMany extends BuilderEntityFilterWithToggle {
```

## Architecture & Concepts
BuilderEntityFilterMany is an abstract base class that forms the cornerstone of the composite filter system for NPC entity targeting. It embodies the *Composite* design pattern, enabling the creation of complex logical filter groups from simpler, individual filters.

Its primary architectural function is to act as a factory for `IEntityFilter` objects that represent a collection of other filters. Concrete implementations, such as `BuilderEntityFilterAllOf` (logical AND) or `BuilderEntityFilterAnyOf` (logical OR), extend this class to define the specific combination logic.

During the asset loading phase, when the server parses an NPC configuration file, this builder is responsible for interpreting a JSON array of filter definitions. It delegates the complex task of parsing, validating, and managing this list of sub-builders to its internal `BuilderObjectListHelper`. This separation of concerns keeps the logic clean: BuilderEntityFilterMany handles the concept of "a filter made of many filters," while BuilderObjectListHelper manages the mechanics of the list itself.

### Lifecycle & Ownership
-   **Creation:** An instance of a concrete subclass is created on-demand by the `BuilderManager` service. This occurs whenever a corresponding filter type (e.g., "allOf") is encountered within an NPC's JSON configuration file during server startup or asset hot-reloading.
-   **Scope:** The lifecycle of a BuilderEntityFilterMany instance is extremely brief and tied directly to a single parsing operation. It is a transient, single-use object that exists only to process one specific JSON array.
-   **Destruction:** The instance is immediately eligible for garbage collection after its `build` method is called and the final, composite `IEntityFilter` object has been returned to the asset system. It holds no references and maintains no state beyond the scope of the build process.

## Internal State & Concurrency
-   **State:** This object is highly stateful but only during its short lifecycle. Its primary state is the list of child filter builders contained within the `objectListHelper` field. This state is progressively built as the `readConfig` method processes the input JSON.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be instantiated and used by a single thread within the server's asset loading pipeline. Concurrent modification of the internal `objectListHelper` would result in a corrupted or invalid filter object. The asset loading framework must enforce a single-threaded execution context for each builder instance.

## API Surface
The public API is minimal, intended for use only by the `BuilderManager` and validation systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement) | Builder<IEntityFilter> | O(N) | Parses a JSON array, delegating the creation of N sub-builders to the objectListHelper. Throws a configuration error if the input is not an array. |
| validate(...) | boolean | O(N\*M) | Recursively validates the entire filter tree. Pushes a new context to the validation helper to detect circular dependencies before delegating to its N children, each with an average validation complexity of M. |
| registerTags(Set<String>) | void | O(1) | Registers the "logic" tag with the asset system, allowing for introspection and categorization of this builder type. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is an internal component of the asset loading system. A content designer defines a filter group in a JSON configuration, and the engine uses a concrete implementation of this builder to realize it.

```json
// Example NPC Configuration Snippet
"targetFilter": {
  "type": "allOf", // This type string maps to a concrete subclass of BuilderEntityFilterMany
  "filters": [     // The readConfig method processes this array
    {
      "type": "isHostile"
    },
    {
      "type": "inRange",
      "range": 20
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new`. The `BuilderManager` is solely responsible for the lifecycle of all builder objects. Direct instantiation bypasses critical dependency injection and context setup.
-   **State Reuse:** Do not attempt to cache and reuse a builder instance to parse multiple JSON objects. Each instance is stateful and must be discarded after a single use.
-   **Concurrent Modification:** Do not access a builder instance from multiple threads. The asset loading pipeline guarantees serial processing, and violating this assumption will lead to unpredictable behavior and data corruption.

## Data Pipeline
This builder acts as a crucial transformation step, converting declarative configuration data into executable game logic.

> Flow:
> NPC_Config.json -> Gson Parser -> BuilderManager -> **BuilderEntityFilterMany** -> Composite IEntityFilter (in-memory object) -> NPC Targeting System

