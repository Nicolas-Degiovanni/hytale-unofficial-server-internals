---
description: Architectural reference for ValueStoreValidator
---

# ValueStoreValidator

**Package:** com.hypixel.hytale.server.npc.valuestore
**Type:** Transient

## Definition
```java
// Signature
public class ValueStoreValidator {
```

## Architecture & Concepts
The ValueStoreValidator is a critical diagnostic tool used during the server-side NPC asset compilation and loading process. Its primary responsibility is to enforce data ownership and prevent logical conflicts within an NPC's internal state, which is managed by a ValueStore.

This class functions as a static analysis engine for NPC behavior definitions. Before an NPC is fully instantiated, its constituent components (behaviors, AI routines, state machines) declare their intended interactions with the shared ValueStore—specifically, which values they intend to write. The ValueStoreValidator collects these declarations and validates them against a core rule: **a value cannot have multiple writers if at least one writer claims exclusive ownership.**

By detecting these conflicts at build-time, the system prevents subtle and difficult-to-debug runtime errors, such as one AI behavior overwriting the calculations of another. It is a safeguard that ensures the integrity and predictability of complex, multi-component NPC logic.

## Lifecycle & Ownership
- **Creation:** A ValueStoreValidator is instantiated by a high-level asset compiler or builder, typically at the beginning of the validation process for a single NPC definition. It is not a globally shared service.

- **Scope:** The object's lifetime is strictly bound to the validation of one composite asset. It accumulates state via `registerValueUsage` calls as the asset's sub-components are processed.

- **Destruction:** The instance is intended to be short-lived. Once the `validate` method is called and its results are consumed, the object holds no external resources and is eligible for garbage collection. It should not be reused for subsequent validations.

## Internal State & Concurrency
- **State:** The ValueStoreValidator is highly stateful and mutable. Its internal `usages` map is populated cumulatively with every call to `registerValueUsage`. The state represents the complete data access map for the asset under validation.

- **Thread Safety:** This class is **not thread-safe**. It is designed for use within a single, synchronous build thread. The internal collections are not synchronized, and concurrent modification will lead to data corruption and incorrect validation results. Do not share an instance across multiple threads.

## API Surface
The public contract is minimal, designed to support a simple register-then-validate workflow.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerValueUsage(ValueUsage) | void | O(1) | Registers a component's intent to write to a specific value in the ValueStore. This method is the primary mechanism for populating the validator's internal state. |
| validate(List<String>) | boolean | O(N) | Executes the validation logic across all registered usages. Returns true if no conflicts are found. If false, the provided list is populated with detailed error messages. N is the total number of registered write usages. |

## Integration Patterns

### Standard Usage
The ValueStoreValidator is designed to be used as part of a Builder or Factory pattern during asset loading. A central builder creates an instance and passes it down to sub-builders, which register their value usages. Finally, the central builder triggers the validation.

```java
// A high-level NPC builder orchestrates the validation
List<String> validationErrors = new ObjectArrayList<>();
ValueStoreValidator validator = new ValueStoreValidator();

// Delegate to sub-builders, passing the same validator instance
healthComponentBuilder.registerValueUsages(validator);
aiBehaviorBuilder.registerValueUsages(validator);
combatLogicBuilder.registerValueUsages(validator);

// Perform the final validation check
boolean isNpcDefinitionValid = validator.validate(validationErrors);
if (!isNpcDefinitionValid) {
    // Abort the build process and log the errors
    throw new AssetCompilationException(String.join("\n", validationErrors));
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not reuse a ValueStoreValidator instance to validate a second, unrelated asset. The internal state is cumulative and will produce incorrect results by mixing usages from both assets.

- **Concurrent Registration:** Do not call `registerValueUsage` from multiple threads on a shared instance. This will cause a `ConcurrentModificationException` or silent data corruption. The entire validation for a single asset must be synchronous.

- **Ignoring Read-Only Usages:** The validator intentionally ignores `READ` operations to focus solely on write conflicts. Do not attempt to modify it to track reads, as this is outside its designed scope of enforcing write ownership.

## Data Pipeline
The validator operates on configuration and definition data, not live game data. Its pipeline is a core step in the asset-to-entity compilation flow.

> Flow:
> NPC Definition File -> Asset Parser -> Component Builders -> `registerValueUsage` -> **ValueStoreValidator** -> `validate()` -> Error List / Validated Definition

