---
description: Architectural reference for TagSetExistsValidator
---

# TagSetExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility / Flyweight

## Definition
```java
// Signature
public class TagSetExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The TagSetExistsValidator is a concrete implementation of the **Strategy** pattern, specializing the generic AssetValidator contract. Its sole responsibility is to confirm that a given string identifier corresponds to a valid, registered NPC group tag set.

This class acts as a critical guard within the server's asset loading and validation pipeline. When the server parses NPC definition files, this validator is invoked to ensure that references to NPC groups—such as faction, behavior, or classification tags—are not broken. It decouples the low-level validation logic from the higher-level asset parser, allowing validation rules to be composed and applied declaratively.

The validator's source of truth is the static asset map maintained by the NPCGroup class. By querying this central registry, it guarantees consistency across all NPC assets.

### Lifecycle & Ownership
-   **Creation:** Instantiation is strictly controlled via static factory methods. Direct construction is prohibited by a private constructor.
    -   The `required()` method provides a shared, static, default instance (Flyweight pattern), minimizing object allocation for the most common use case.
    -   The `withConfig()` method constructs a new instance for specialized validation scenarios that require non-default configuration.
-   **Scope:** The default singleton instance is application-scoped and persists for the entire server lifetime. Instances created via `withConfig` are typically transient, owned by an asset builder or schema definition, and live only for the duration of the asset loading process.
-   **Destruction:** Custom-configured instances are eligible for garbage collection once the owning asset builder is destroyed. The static default instance is never collected.

## Internal State & Concurrency
-   **State:** **Immutable**. An instance of TagSetExistsValidator holds no mutable state. Its configuration is provided at construction and cannot be changed. The outcome of its `test` method is entirely dependent on the external, static state of the `NPCGroup` asset map.

-   **Thread Safety:** **Conditionally Thread-Safe**. The class itself contains no locks or mutable fields, making it safe for concurrent access. However, its correctness depends on the thread safety of the external `NPCGroup.getAssetMap()`.

    **WARNING:** This validator assumes that the `NPCGroup` asset map is populated during a single-threaded server bootstrap phase and is subsequently treated as a read-only collection. Concurrent modification of the `NPCGroup` map while validation is in progress will result in race conditions and non-deterministic behavior.

## API Surface
The public API is minimal, focusing on object creation and the implementation of the AssetValidator contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| required() | TagSetExistsValidator | O(1) | Factory method. Returns the shared, default singleton instance. |
| withConfig(config) | TagSetExistsValidator | O(1) | Factory method. Creates a new validator with specific configuration. |
| test(value) | boolean | O(1) | Executes the validation logic. Returns true if the value exists in the NPCGroup asset map. Assumes the underlying map provides constant-time lookups. |
| errorMessage(value, attribute) | String | O(1) | Generates a formatted, human-readable error message for validation failures. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly in game logic. It is designed to be plugged into a declarative asset definition or schema builder during server initialization. The asset loading framework then invokes it automatically.

```java
// Hypothetical usage within an Asset Schema Definition

// The framework consumes this definition and applies the validator
// to the "npcGroup" attribute of a parsed asset file.
AssetSchemaBuilder.create("CustomNPC")
    .withAttribute("npcGroup", String.class)
    .addValidator(TagSetExistsValidator.required())
    .build();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to call `new TagSetExistsValidator()`. The constructor is private. Always use the `required()` or `withConfig()` static factory methods.
-   **Misuse for Other Asset Types:** This validator is hard-coded to check against the `NPCGroup` registry. Attempting to use it to validate other types of assets, such as item tags or block types, will always fail or produce incorrect results.
-   **Assuming Dynamic Updates:** Do not expect the validator to recognize NPC groups added at runtime after the initial server bootstrap. The validator reads from a static map that is assumed to be finalized on startup.

## Data Pipeline
The TagSetExistsValidator functions as a gate in the data flow of asset processing. It does not transform data; it either permits it to pass or halts the process by raising a validation error.

> Flow:
> NPC Asset File (e.g., JSON) -> Server Asset Parser -> **TagSetExistsValidator.test(value)** -> [Success] Asset Instantiated OR [Failure] Validation Error Logged & Asset Rejected

