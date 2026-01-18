---
description: Architectural reference for BuilderEntityFilterBase
---

# BuilderEntityFilterBase

**Package:** com.hypixel.hytale.server.npc.corecomponents.builders
**Type:** Transient Component

## Definition
```java
// Signature
public abstract class BuilderEntityFilterBase extends BuilderEntityFilterWithToggle {
```

## Architecture & Concepts
BuilderEntityFilterBase is an abstract foundational component within the server's NPC (Non-Player Character) definition and loading pipeline. It is not a standalone service but rather a building block for creating specific entity filtering rules.

Its primary architectural role is to enforce configuration sanity and prevent logical conflicts during the load-time validation of an NPC's definition. Specifically, it ensures that any given entity filter type is defined only once and is not made redundant by an externally provided system, such as a "prioritiser". This prevents ambiguous or conflicting behavior in how an NPC selects targets or interacts with other entities.

This class embodies the "fail-fast" principle for server content loading. By catching these specific configuration errors before an NPC is fully instantiated, it guarantees that any loaded NPC has a clear, unambiguous set of filtering rules, preventing difficult-to-diagnose runtime bugs.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses are instantiated by the NPC configuration parsing system when it encounters a filter definition within an NPC's configuration file. This class is never instantiated directly.
- **Scope:** The object's lifecycle is exceptionally brief. It exists only for the duration of the validation step for a single NPC configuration.
- **Destruction:** The instance is eligible for garbage collection immediately after the `validate` method returns. It holds no references to long-lived services and maintains no persistent state, ensuring a minimal memory footprint during content loading.

## Internal State & Concurrency
- **State:** This component is effectively stateless. While it reads its own configuration (set during parsing), its core `validate` method does not mutate its own internal state. However, it operates upon and modifies the state of external objects passed to it, namely the `NPCLoadTimeValidationHelper` and the `errors` list.

- **Thread Safety:** This class is **not thread-safe** and must not be used in a concurrent context. The validation process it belongs to is designed to be a single-threaded, sequential operation during server startup or content hot-swapping. The `NPCLoadTimeValidationHelper` object that it interacts with is a stateful tracker for a single validation session. Concurrent calls to `validate` using the same helper instance would result in a race condition, leading to incorrect validation results and corrupted error logs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validate(configName, validationHelper, context, globalScope, errors) | boolean | O(1) | Executes uniqueness validation for the filter type. Returns `false` if the filter is a duplicate or is superseded by an external provider, populating the `errors` list with details. |

## Integration Patterns

### Standard Usage
This class is intended to be extended, not used directly. A concrete implementation provides a specific filter type, and the engine's validation pipeline invokes its `validate` method.

```java
// A concrete filter builder extending the base class
public class MySpecificFilterBuilder extends BuilderEntityFilterBase {
    @Override
    public String getTypeName() {
        return "my_specific_filter";
    }
    // ... other implementation details
}

// The validation system (conceptual)
public void validateNpcConfiguration(NpcConfig config, List<String> errorList) {
    NPCLoadTimeValidationHelper validationHelper = new NPCLoadTimeValidationHelper();
    // ...
    for (BuilderEntityFilterBase filter : config.getFilters()) {
        boolean isValid = filter.validate(config.getName(), validationHelper, context, scope, errorList);
        if (!isValid) {
            // Halt loading or log critical errors
            return;
        }
    }
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Return Value:** The boolean result from `validate` is critical. Ignoring it means that a failed validation will not halt the NPC loading process, potentially leading to undefined or erroneous NPC behavior at runtime. Always check the return value.
- **Stateful Subclasses:** Subclasses should not introduce state that persists beyond the scope of a single `validate` call. Doing so violates the stateless, transient nature of the component and could introduce memory leaks or state corruption between different NPC validations.
- **Reusing Validation Helper:** The `NPCLoadTimeValidationHelper` instance is stateful and scoped to the validation of a *single* NPC configuration. Reusing the same helper instance across validations of different, unrelated NPC configurations without resetting it will cause incorrect "duplicate filter" errors.

## Data Pipeline
The component acts as a validation stage within a larger data transformation pipeline that converts static configuration files into live game objects.

> Flow:
> NPC Definition File (JSON/YAML) -> Configuration Parser -> **Concrete Subclass of BuilderEntityFilterBase** -> Uniqueness & Redundancy Check -> Error List or Validated State -> NPC Instantiation

